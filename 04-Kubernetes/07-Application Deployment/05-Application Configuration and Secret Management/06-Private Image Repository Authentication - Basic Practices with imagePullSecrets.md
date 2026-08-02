# 06-Private Image Repository Authentication: Basic Practices with imagePullSecrets

## Document Description
- Document Purpose: Introduction to private image repository pull authentication practices
- Applicable Phase: After completing basic steps such as image building, pushing, and Deployment setup, proceed to learning about private repository authentication and image pulling processes
- Recommended Path: `04-Kubernetes/07-Application Deployment/05-Application Configuration and Key Management/06-Private Image Repository Authentication: Basic Practices with imagePullSecrets`

## Tags
#Kubernetes #Secret #imagePullSecrets #Harbor #Private Image Repository #Image Pull #ErrImagePull #ImagePullBackOff #containerd #Docker #Configuration Management #Application Deployment #Business Containerization #Cloud Native #Operations and Maintenance #Interview Notes

---

## I. Why Is It Necessary to Learn about imagePullSecrets

You have already completed the following basic steps:

- Building images using Dockerfile
- Naming and tagging images
- Pushing images to a repository
- Starting Pods using these images via Deployment

However, there is a practical issue here:

**If the images are stored in a private repository, how can nodes pull them down?**

In a local environment, many people overlook this problem because they may have already performed the following actions:

- `docker login`
- Or the local machine may have cached the images

But Kubernetes nodes do not automatically inherit the local login state. For a node, if it lacks appropriate repository authentication information, it is likely to encounter issues such as:

- `ErrImagePull`
- `ImagePullBackOff`

These problems are very common in real-world environments and are also one of the frequent causes of application deployment troubleshooting.

Therefore, learning about `imagePullSecrets` is essential because it ensures that Kubernetes has legitimate authentication credentials when it needs to pull private images.

---

## II. What Problem Does imagePullSecrets Solve

It solves the problem of:

**How Pods can use the repository authentication information stored in Kubernetes to pull private images during startup.**

In other words, `imagePullSecrets` is not a Secret designed for business applications but rather for Kubernetes to use when pulling images.

### You Can Understand It This Way

- Ordinary Secrets: Read by business programs
- imagePullSecrets: Read by Kubernetes when pulling images

---

## III. Why Does it Also Belong to the Secret System

Because repository authentication information is also sensitive data, such as:

- Repository username
- Repository password
- Access Token
- Docker config json

Clearly, this type of information should not be hardcoded in:

- Deployment YAML files
- Images
- Plain-text scripts

Therefore, Kubernetes uses Secrets to store it.

In other words:

**The underlying mechanism of imagePullSecrets is still based on Secrets; its purpose is merely for image pull authentication.**

---

## IV. In What Scenarios Is imagePullSecrets Required?

The following scenarios typically require the use of `imagePullSecrets`:

### 1. Images are stored in a private Harbor

For example, accessing an internal enterprise Harbor project requires login.

### 2. Images are stored in cloud vendors' private repositories

For example:

- Alibaba Cloud ACR
- Tencent Cloud TCR
- Huawei Cloud SWR
- AWS ECR
- GCR private images

### 3. Nodes have not previously logged into the repository

Even if the local machine has logged in, the nodes themselves may not have done so.

### 4. The cluster is a multi-node environment

Just because one node has pulled an image before does not mean that other nodes have authentication information.

---

## V. When Is imagePullSecrets Usually Not Needed?

### 1. Using public image repositories

For example, directly pulling public images such as:

- `nginx`
- `busybox`
- `redis`

### 2. Nodes already have available authentication locally

In some environments, repository authentication is configured at the node level in advance, but this approach is not as standardized as managing it within Kubernetes.

### 3. Using a local development single-node environment where images are pre-loaded

For example, in certain local experimental environments, images are already imported into the nodes.

### Key Points for Operations and Maintenance Professionals to Understand

From a regulatory perspective, for any formal private image usage scenario, it is recommended to prioritize using Kubernetes to officially manage repository pull credentials through Secrets.

---

## VI. Understanding Two Common Error States When Pulling Images Before Learning about imagePullSecrets

Before delving into `imagePullSecrets`, it is important to understand two common error messages:

### 1. `ErrImagePull`

This indicates that an error occurred while the node was attempting to pull the image.

### 2. `ImagePullBackOff`

This indicates that the node entered a backoff and retry state after failing to pull the image.

### Common Causes for These Two States Include

- Incorrect repository address
- Incorrect image name
- Non- `aliyun-registry-secret`
  This indicates the name of the Secret, which will be referenced within the Pod.

- `--namespace=test`
  This specifies that the Secret is created in the `test` namespace.

- `--docker-server=registry.cn-hangzhou.aliyuncs.com`
  This indicates the address of the image repository.

- `--docker-username=syq1013`
  This indicates the username for accessing the repository.

- `--docker-password=<REGISTRY_PASSWORD>'`
  This indicates the password or access credentials for the repository. The placeholder is used only for illustrative purposes.

- `--docker-email=example@example.com`
  This is a compatible field that can generally be retained.```json
{
  "auth": "<BASE64_OF_USERNAME_COLON_PASSWORD>"
}
```

The most easily misunderstood point here is:

**`auth` does not involve Base64 encoding the password alone, but rather the entire `username:password` pair.**

For example:

- Username: `your-registry-user`
- Password: `<REGISTRY_PASSWORD>`

The correct approach is to encode the following content:

    your-registry-user:<REGISTRY_PASSWORD>

---

## 21. Why You Can’t Use `echo` for Base64 Encoding Arbitrarily

Many people simply execute the following when generating Base64:

    echo "example-password" | base64

Or:

    echo -e "example-password" | base64

This method usually includes **line breaks** in the encoded result.  
In other words, the actual encoded content is not:

    example-password

but rather:

    example-password\n

This can lead to incorrect Base64 results.  
If this error occurs in `auth` or Secret data, it may cause image authentication failures.

### Key Points for Operations Engineers

In scenarios involving image repository authentication, Tokens, Secrets, and certificate processing, **even a single line break can result in authentication failure**.

---

## 22. The Correct Way to Encode Base64

If you only need to verify a single string, use:

    echo -n 'example-password' | base64

The `-n` option ensures that **no line break is appended at the end**.

---

## 23. The Proper Way to Generate the `auth` Field

To create the `auth` field commonly used in Docker repository authentication, you need to encode the entire `username:password` pair:

For example:

- Username: `your-registry-user`
- Password: `example-password`

The correct command is:

    echo -n 'your-registry-user:example-password' | base64

This will generate a result suitable for use in:

    "auth": "..."

### Key Points for Operations Engineers

It’s important to distinguish between these two scenarios:

#### 1. Encoding the password alone

    echo -n 'example-password' | base64

This only encodes the **password string itself**.

#### 2. Encoding `username:password` together

    echo -n 'your-registry-user:example-password' | base64

This is the format required for the `auth` field in `.dockerconfigjson`.

---

## 24. How to Verify If a Base64 String Is Correct

If you have a Base64 string, you can decode it using:

    echo 'BASE64-content' | base64 -d

Or:

    echo 'BASE64-content' | base64 --decode

This will reveal the original content.

---

## 25. How to Check for Hidden Line Breaks in Decoded Results

Sometimes, the decoded content may seem fine on the surface, but it might actually contain hidden line breaks at the end.  
To check this, use:

    echo 'BASE64-content' | base64 -d | cat -A

Or:

    echo 'BASE64-content' | base64 -d | xxd

### Explanation

- `cat -A` is useful for quickly checking for hidden characters.
- `xxd` provides a hexadecimal view of the content, allowing you to identify invisible characters like line breaks and spaces.

These tools are particularly helpful when troubleshooting authentication fields, Tokens, password strings, and certificate contents.

---

## 26. Why `imagePullSecrets` Is Defined in Pod Templates

Because the actual act of pulling images occurs at the Pod level, not at the Deployment level.

### What a Deployment Does

- Manages replicas.
- Handles Pod templates.
- Performs rolling updates.

### What a Pod Template Does

- Defines container images.
- Sets environment variables.
- Configures volumes.
- Specifies `imagePullSecrets`.

### Key Points for Operations Engineers

Deployments themselves do not pull images; instead, the nodes where Pods run are responsible for this task.  
Therefore, repository authentication settings must be included in the **Pod template spec**.

---

## 27. Example 2: If It’s an Enterprise Private Harbor, Are the Commands and YAML Formats the Same?

**Essentially, they are the same.**

You just need to replace the following values:

- `--docker-server` with the enterprise Harbor address.
- `--docker-username` with the Harbor username.
- `--docker-password` with the Harbor password.
- `image` with the Harbor image path.
- The Secret name can be adjusted according to your company’s standards.

For example, if the enterprise Harbor address is:

    harbor.example.com

and the image path is:

    harbor.example.com/devops/nginx-web:v1,

the namespace remains:

    test,

and the local login command## 32. Additional Notes: What to Do If Harbor Uses Internal HTTP Protocol

Let's start with a very important premise:

`imagePullSecrets` can only address authentication issues; it cannot enable nodes to automatically accept an HTTP repository.

If Harbor does not use HTTPS at all but instead uses:

    http://harbor.example.com

for the repository, for the container runtime on the node, this would be considered:

- Plain-text HTTP Registry
- Insecure Repository

Even if you have correctly configured:

- Harbor username and password
- `imagePullSecrets`
- The private image address in the Deployment

the node may still be unable to pull images.

Because the issue here is no longer "whether there is authentication information" but rather:

**Whether the container runtime on the node allows HTTP access to this repository.**

### Key Understanding for This Scenario

If Harbor is an HTTP repository, what needs to be addressed is not the Pod YAML itself but rather:

**The configuration of the container runtime on each Kubernetes node.**

In other words, you need to ensure that:

- Docker
- containerd

on the node level recognize Harbor as an "accessible HTTP repository."

### Key Points for Operations and Maintenance

This is a completely different issue from `imagePullSecrets`:

#### 1. What `imagePullSecrets` does:
- Verifies whether the account password is correct.
- Checks if there is authentication information for private images.

#### 2. What node trust of HTTP Harbor addresses:
- Whether the container runtime allows HTTP access to the repository.
- Whether it treats the repository as an Insecure Registry.

So, if Harbor uses HTTP, you need to ensure that:

1. The container runtime on the node allows access to this HTTP repository.
2. The Pod has correct authentication information when pulling private images.

Otherwise, images will still not be able to be pulled.

---

## 33. How to Trust an Internal HTTP Harbor If the Node Uses Docker

If the container runtime on the node is Docker, a common approach is to modify the following file on the node:

    /etc/docker/daemon.json

For example:

    {
      "insecure-registries": ["harbor.example.com"]
    }

If Harbor uses a port, you also need to include that port, like this:

    {
      "insecure-registries": ["harbor.example.com:8080"]
    }

After making these changes, restart Docker:

    systemctl daemon-reload
    systemctl restart docker

Then, you can check by running:

    docker info

To see if it now displays:

    Insecure Registries

If you can successfully log in to Harbor using `docker login harbor.example.com` and pull an image like `docker pull harbor.example.com/devops/nginx-web:v1` without any issues, it means that Docker on this node has accepted Harbor as an HTTP/Insecure Repository.

### Key Points for Operations and Maintenance

If a Kubernetes node uses Docker as the runtime, this configuration is at the **node level** and not part of the cluster configuration.

In other words:

- Changing a Deployment will not automatically make nodes trust the HTTP Harbor.
- Changing a Secret will also not have the same effect.
- The configuration must be applied directly to each node that needs to pull images.

---

## 34. How to Trust an Internal HTTP Harbor If the Node Uses containerd

In the context of containerd, the general approach is:

- Create a `registry host` directory for the specified Harbor address.
- Place a `hosts.toml` file in this directory.
- Clearly tell containerd that this Harbor should be accessed via HTTP.

### Two Important Points to Note First

#### 1. The main difference between containerd 1.x and 2.x lies in the `config.toml` file
For containerd 1.x, the common registry configuration path is usually:

    [plugins."io.containerd.grpc.v1.cri".registry]

For containerd 2.x, it is usually:

    [plugins.'io.containerd.cri.v1.images'.registry]

This means that while the plugin paths in `config.toml` differ between versions 1.x and 2.x, the process of setting up HTTP trust for a Harbor often follows the same general principle, which involves modifying the `certs.d/<registry>/hosts.toml` file.

#### 2. The following content is based on the current environment
If the node is using **containerd 2.x** and the `config.toml` file already specifies the registry configuration path, for example:

    version = 3

    [plugins.'io.containerd.cri.v1.images'.registry]
      config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'

This means that the system will look for the registry host configuration in the following locations:

- `/etc/containerd/certs.d`
- `/etc/docker/certs.d`

However, for consistency and maintain```bash
kubectl get secrets -n <namespace>
```

### 2. 查看 Secret 的命名空间是否正确

```bash
kubectl get secrets -n <namespace> --name <secret-name>
```

### 3. 查看镜像地址和 Secret 认证内容是否匹配

```bash
kubectl get images -n <namespace> --image=<image-name>
kubectl get secrets -n <namespace> --name=<secret-name>
```

### 4. 验证 imagePullSecrets 是否正确配置

```bash
kubectl get secrets -n <namespace> --name=<secret-name>
kubectl get pods -n <namespace> --selector=app=<application-name> --image=<image-name>
```

### 5. 查看容器运行时配置

```bash
kubectl get daemonconfig -n <node-name>
kubectl config get containerd --show-file=/etc/containerd/config.toml
```

### 6. 验证仓库证书问题

```bash
curl -I https://<warehouse-url> --cert-file=/path/to/warehouse-ca-cert.pem
```

By following these steps and using the appropriate commands, you can effectively troubleshoot image pull failures in Kubernetes.### 2. View Secret Details

    kubectl describe secret aliyun-registry-secret -n test

### 3. Check Pod Status

    kubectl get pod -n test

### 4. View Pod Detailed Information and Events

    kubectl describe pod -n test <pod-name>

### 5. Review Deployment Configuration

    kubectl get deploy nginx-web -n test -o yaml

### 6. View the Original YAML of Secret

    kubectl get secret aliyun-registry-secret -n test -o yaml

### 7. Examine containerd Configuration

    cat /etc/containerd/config.toml

### 8. List Harbor's Corresponding Registry Directories

    ls -R /etc/containerd/certs.d

### 9. Verify Image Pull Using CRI Tools

    crictl pull harbor.example.com/devops/nginx-web:v1

---

## Forty-Five, Key Understandings in This Chapter

### 1. Private image repositories usually require authentication
Nodes by default do not have inherent access rights.

### 2. Kubernetes uses Secrets to store repository authentication information
This is part of sensitive configuration management.

### 3. `imagePullSecrets` are used for Pod image pulls
They are not meant for business programs to read.

### 4. `imagePullSecrets` itself is not an independent object
It is merely a way Pods/Deployments reference Secrets.

### 5. The actual authentication information is stored in a `Secret`
Typically, the Secret type is:

    kubernetes.io/dockerconfigjson

### 6. YAML configuration typically consists of two parts:
- `Secret YAML`
- `Deployment YAML`

### 7. `auth` is not just Base64-encoded password
It is Base64-encoded `username:password`.

### 8. By default, `echo` may include line breaks
Without `-n`, the encoded result might not be as expected.

### 9. Harbor and cloud vendors' private repositories are essentially similar in creation
Differences mainly lie in repository addresses, account systems, and certificate environments.

### 10. Harbor may face three types of issues:
- Authentication problems
- HTTP repository trust issues
- HTTPS self-signed certificate trust issues

### 11. The main difference between containerd 1.x and 2.x lies in `config.toml`
- In 1.x, it is commonly `plugins."io.containerd.grpc.v1.cri".registry`
- In 2.x, it is commonly `plugins.'io.containerd.cri.v1.images'.registry`

### 12. If the current environment uses containerd 2.x:
Ensure:
- `version = 3`
- `config_path`
- `/etc/containerd/certs.d/<registry>/hosts.toml`

### 13. Do not include actual passwords in knowledge bases or examples
Use placeholders instead, such as:

- `<REGISTRY_PASSWORD>`
- `<BASE64_OF_USERNAME_COLON_PASSWORD>`
- `<HARBOR_PASSWORD>`

---

## Forty-Six, What Level of Mastery Is Required for This Chapter

At this stage, it is recommended to achieve the following levels:

### 1. Understand why private repositories require `imagePullSecrets`
### 2. Comprehend that `imagePullSecrets` essentially depend on Secrets
### 3. Be able to interpret how `imagePullSecrets` are defined in Deployments
### 4. Distinguish between `imagePullSecrets` and regular business Secrets
### 5. Make initial judgments when image pulls fail
### 6. Create `imagePullSecrets` using commands
### 7. Understand the relationship between Secrets and Deployments in dual-file versions
### 8. Recognize similarities and differences between Harbor and cloud vendors' private repositories
### 9. Grasp how `auth` is generated and common Base64 pitfalls
### 10. Differentiate between HTTP, HTTPS self-signed certificates, and authentication issues in Harbor
### 11. Understand the differences in trust mechanisms between Docker and containerd for Harbor
### 12. Note that the registry configuration path differs between containerd 1.x and 2.x, while Harbor host trust typically uses `certs.d`
### 13. Avoid writing actual passwords directly into command history or version control

---

## Forty-Seven, Common Extended Questions in Interviews

Frequently asked questions include:

- How are private image repositories pulled in Kubernetes?
- What is `imagePullSecrets`?
- What's the difference between `imagePullSecrets` and regular Secrets?
- Why does a Pod encounter `ImagePullBackOff`?
- How to troubleshoot general image pull failures?
- Why are Secrets also used for image pull authentication?
- Where should `imagePullSecrets` be defined?
- Why can images be pulled locally but not in the cluster?
- How to create a `- Post-deployment troubleshooting

Looking deeper into this topic, it is also connected to several very practical issues:

- How to securely store credentials
- Whether passwords will be logged in the history
- Whether Secrets might be accidentally committed to the repository
- Whether enterprise Harbor uses HTTP
- Whether the certificates of enterprise Harbor are trusted by nodes
- Whether the `config_path` of containerd has been correctly specified
- Whether `hosts.toml` is configured correctly
- Whether authentication failures occur due to line breaks, spaces, or formatting errors during the encoding process

Therefore, although this topic may seem minor, it is actually a crucial step in moving from "being able to deploy YAML files" to "understanding the actual delivery process of images".

---

## References
- Kubernetes Images: https://kubernetes.io/docs/concepts/containers/images/
- Pulling an Image from a Private Registry: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/

---

## Suggestions for the Next Day
It is recommended to organize the following topic in the next article:

[[01-livenessProbe and readinessProbe Basic Practices]]