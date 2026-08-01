# 03-Image Naming, Tag, Building, Pushing, and Pulling Basics

## Document Notes
- Document Positioning: Image Lifecycle Basics
- Applicable Stage: Business Containerization Learning Fourth Article / Image Management and Distribution Basics
- Recommended Path: `04-Kubernetes/07-Apply deployment/01-Mirror and container base/03-Mirror Naming,tag, build, push and pull the base`

## Tags
#Kubernetes #Docker #Mirror #MirrorRepository #tag #MirrorBuild #MirrorDelivery #MirrorPull #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Learn Image Naming, Tag, Building, Pushing, and Pulling

In the process of business containerization, Dockerfile solves:

**How to create an image.**

While image naming, tag, building, pushing, and pulling solve:

**How to identify, manage, distribute, and use images.**

If this part is unclear, you'll easily encounter the following issues in Kubernetes deployment:

- The image name is correct, but the node cannot pull it
- The tag is written as `latest`, but the deployed version is not the expected one
- The image has been rebuilt, but the Pod hasn't updated
- Different naming conventions are used in development, testing, and production environments, making version tracking difficult
- Private registry authentication fails, causing the Pod to always `ImagePullBackOff`
- The same image is repeatedly built and pushed by multiple people, leading to version chaos

Therefore, this section is a critical step from "knowing how to write Dockerfile" to "managing the image delivery pipeline."

---

## II. Basic Lifecycle of an Image

From business application to Kubernetes runtime, images typically go through the following process:

### 1. Writing Dockerfile
Define how to build the image.

### 2. Building the Image Locally
Build the image from Dockerfile and business files locally.

### 3. Naming and Tagging the Image
Give the image a clear identity and version.

### 4. Pushing to Image Repository
Allow other machines and Kubernetes nodes to pull the image.

### 5. Kubernetes Node Pulling the Image
Pull the image from the repository based on Deployment, StatefulSet, etc.

### 6. Starting the Container
Start the business process via the container runtime.

This pipeline can be summarized as:

**Business Files → Dockerfile → Build Image → Naming and Tagging → Push to Repository → Node Pulling → Start Container**

---

## III. Basic Structure of Image Naming

A complete image name typically contains the following parts:

    Repository Address/Namespace/Image Name:tag

Example:

    harbor.example.com/project/nginx-web:v1.0.0

This can be understood as four parts.

### 1. Repository Address
Examples:
- `docker.io`
- `harbor.example.com`
- `registry.example.com`

Indicates where the image is stored in the repository service.

### 2. Namespace or Project Name
Examples:
- `library`
- `project`
- `devops`

Used for classification, isolation, and permission management.

### 3. Image Name
Examples:
- `nginx-web`
- `user-service`
- `nacos-server`

Represents the specific name of the business image.

### 4. Tag
Examples:
- `v1.0.0`
- `20260401`
- `dev-001`
- `latest`

Represents the version or identifier of the image.

---

## IV. What is a Tag

A tag can be understood as:

**The version label of an image.**

Its purpose is not to modify the image content, but to assign an identifiable version name to a specific image content.

Examples:
    app:v1.0.0
    app:v1.0.1
    app:dev
    app:latest

Here, `v1.0.0`, `v1.0.1`, `dev`, and `latest` are all tags.

---

## V. Why Tags Are Important

### 1. For Version Differentiation
Different tags usually correspond to different versions of the image.

### 2. For Releases and Rollbacks
Examples:
- Production environment currently uses `v1.0.0`
- Upgrade to `v1.0.1`
- Rollback to `v1.0.0` when issues occur

### 3. For Environment Differentiation
Examples:
- `dev`
- `test`
- `prod`

### 4. For CI/CD Pipeline Management
CI/CD often generates tags based on:
- Git commit
- Branch name
- Build number
- Release time

---

## VI. Common Tag Naming Conventions

Common tag naming conventions in production include several types.

### 1. Semantic Versioning
Examples:
    v1.0.0
    v1.0.1
    v2.3.5

Advantages:
- Clear semantic versioning
- Facilitates releases and rollbacks
- Suitable for formal versions

### 2. Time-Based Versioning
Examples:
    20260401
    20260401-1200

Advantages:
- Clearly indicates the build time
- Suitable for quick traceability of release batches

### 3. Build Number Versioning
Examples:
    build-101
    build-102

Advantages:
- Facilitates association with the pipeline

### 4. Git Commit Versioning
Examples:
    7f3c1a2
    commit-7f3c1a2

Advantages:
- Facilitates precise traceability of code commits

### 5. Environment or Branch Versioning
Examples:
    dev
    test
    prod
    feature-login

Advantages:
- Facilitates environment differentiation
- Suitable for temporary verification

However, there are obvious drawbacks:
- Less precise versioning
- Difficult to track after multiple overwrites

---

## VII. Why Production Environments Should Avoid Long-Term Reliance on 'latest'

`latest` is the easiest tag to misuse.

### 1. It Does Not Represent "Latest Available Version"
It's just a regular tag named `latest`, which does not inherently guarantee the most stable, correct, or recent content.

### 2. It Hinders Rollbacks
If you always push to `latest`, it's hard to quickly determine the previous stable version when issues occur.

### 3. It Hinders Troubleshooting
When troubleshooting and seeing:
    image: app:latest
It's hard to accurately know which build is actually running.

### 4. It Causes Version Management Chaos
In multi-person collaboration and shared repositories across multiple environments, `latest` easily loses traceability.

### Operations Recommendations
- Development and testing environments can temporarily use `latest`
- Pre-release and production environments are recommended to use specific, traceable tags

---

## VIII. What is Image Building

Image building (build) refers to:

**The process of generating an image based on Dockerfile and build context.**

Common build commands include:
    docker build -t myapp:v1.0.0 .

Here's how to understand it:

- `docker build`:Run the build
- `-t`:Name and tag the image
- `myapp:v1.0.0`:Target image name
- `.`:Current directory as build context

### What is a Build Context
The build context refers to:

**The scope of files Dockerfile can read and copy during the build.**

If your current directory contains:

- Dockerfile
- Application code
- Configuration files
- requirements.txt

They may all be included in the build process.

---

## IX. Core Focus Points During Image Building

### 1. Is the Dockerfile correct?
If the Dockerfile itself has issues, the image build may fail immediately.

### 2. Is the build context reasonable?
If the context is too large, the build will be slower and may include irrelevant files in the image.

### 3. Is the tag clear?
You should specify clear versions during the build, not rely on vague tags.

### 4. Is the build result reusable?
Try to make the same content produce consistent images for easier delivery and troubleshooting.

---

## X. What is Image Pushing

Image pushing (push) refers to:

**Uploading the locally built image to an image registry.**

Example:

    docker push harbor.example.com/project/myapp:v1.0.0

Its significance lies in:

- Local builds are only usable on the machine
- After pushing to the registry, other machines and cluster nodes can pull the image
- Kubernetes typically doesn't use Docker images directly from your local machine unless the node already has the image and pull policies are satisfied

---

## XI. What is Image Pulling

Image pulling (pull) refers to:

**Downloading an image from the registry to the local machine or node.**

Example:

    docker pull harbor.example.com/project/myapp:v1.0.0

In Kubernetes, when a Pod needs to start, the node usually automatically performs a similar pull operation.

### Operations Focus Points
If a node cannot pull an image, common manifestations include:

- Pod Pending
- `ErrImagePull`
- `ImagePullBackOff`

You need to troubleshoot from the registry, network, authentication, image name, and tag directions.

---

## XII. Prerequisites for Image Pushing and Pulling

### 1. Correct registry address
If the registry address is wrong, pushing and pulling will fail.

### 2. Correct image name and tag
Wrong name or tag means the target image cannot be found.

### 3. Repository access permissions
Private repositories often require login authentication.

### 4. Network connectivity
The node must be able to access the registry address.

### 5. Trust in the registry
Some private repositories or HTTPS certificate configurations may cause pulling failures.

---

## XIII. Differences Between Public and Private Registries

### 1. Public Registry
Example: Docker Hub

Features:

- Many public images
- Easy to use
- Convenient to pull open-source base images

### 2. Private Registry
Example: Harbor or enterprise internal registry

Features:

- Suitable for internal business image management
- Supports access control
- Supports project isolation
- Facilitates audit and standardized management

### Common Operational Scenarios
Production environments typically see:

- Base images from public registries
- Business images uniformly pushed to private registries
- Kubernetes pulls business images from private registries

---

## XIV. Basic Logic of Image Pulling in Kubernetes

In Kubernetes, Deployment or Pod configurations usually include:

    image: harbor.example.com/project/myapp:v1.0.0

When a Pod is scheduled to a node, the node checks according to the configuration:

### 1. Check if the image already exists locally
### 2. Determine if a pull is needed based on `imagePullPolicy`
### 3. Fetch the image from the registry
### 4. Start the container after successful pull

### Common `imagePullPolicy`
#### 1. `IfNotPresent`
Pull only if the image is not locally present.  
Commonly seen.

#### 2. `Always`
Always attempt to pull.  
Suitable for scenarios needing frequent tag updates, but more dependent on registry availability.

#### 3. `Never`
Never pull, only use local images.  
Usually for special testing scenarios.

---

## XV. Common Issues in Image Management

### 1. Tag chaos
Manifestations:

- Not knowing which tag is the production version
- Mixed use of images across environments
- Difficulty in rolling back

### 2. Using latest leads to untrackable versions
Manifestations:

- Unable to see the actual build version
- Troubleshooting difficulties
- Uncertain update behavior

### 3. Image has been rebuilt, but Pod hasn't updated
Possible causes:

- Tag hasn't changed
- `imagePullPolicy` is inappropriate
- Deployment hasn't triggered rolling update

### 4. Node cannot pull the image
Possible causes:

- Registry address error
- DNS or network issues
- Authentication failure
- Tag doesn't exist
- Registry certificate anomalies

### 5. Large build context
Manifestations:

- Very slow builds
- Irrelevant files included in the image
- Increased security risks

---

## XVI. More Reasonable Image Management Approaches in Production

### 1. Stable image name, clear tag
Example:

- Keep the image name fixed: `user-service`
- Change the tag with version updates: `v1.0.0`

### 2. Unified naming convention across development, testing, and production
Avoid having completely different naming schemes for each environment.

### 3. Tags should be traceable
Recommend at least being able to trace to:

- Build time
- Build number
- Git commit
- Release version

### 4. Avoid long-term reliance on latest in production
Explicit versions make it easier for release, audit, rollback, and troubleshooting.

### 5. Consolidate private business images in enterprise registry
Facilitates access control and centralized management.

---

## XVII. Understanding "Image Not Updated" from an Operations Perspective

This is a very common issue in production.

### Scenario 1: Re-pushed the same tag
Example still pushing:

    app:latest

But the Deployment configuration hasn't changed, the node may not refresh as expected.

### Scenario 2: Local and registry images are not the same version
Rebuilt locally but not pushed; or pushed but Pod still uses old cache.

### Scenario 3: Deployment hasn't triggered an update
Kubernetes often decides whether to roll out updates based on whether the Pod Template has changed.

### Operations Focus Points
Don't just look at "I built an image," but check the full chain:

- Correct tagging
- Correct pushing
- Deployment referencing the new tag
- Node actually pulling the new image
- Pod successfully restarting

---

## XVIII. What Level of Understanding is Required to Learn This Section

At this stage, reaching the following level is sufficient:

### 1. Be able to explain the basic structure of an image name  
### 2. Be able to understand the role of a tag  
### 3. Be able to explain what build, push, and pull mean  
### 4. Be able to understand how Kubernetes nodes pull images  
### 5. Be able to understand why it's not recommended to long-term depend on latest  
### 6. Be able to preliminarily determine where the issue in the image distribution pipeline occurs  

---

## Nineteen, Common Follow-up Questions in Interviews  

This section commonly includes questions such as:  

- What is the basic structure of an image name  
- What is a tag and what is its role  
- Why is it not recommended to long-term use latest  
- What do build, push, and pull mean  
- How does Kubernetes pull images  
- `imagePullPolicy` What are the types  
- Why is the Pod still using the old version even though the image has been updated  
- How to troubleshoot failed pulls from a private registry  
- How are image tags generally designed in production  

---

## Twenty, Stage Summary  

Image naming, tags, building, pushing, and pulling are key processes that transform an image from a "local build product" into a "standard deliverable usable by a cluster."  

The most important part of this section is not memorizing many commands first, but establishing a clear understanding of the image distribution pipeline:  

- Images need clear names  
- Images need traceable tags  
- Local build is just the first step  
- Images can only be used by other machines and cluster nodes after being pushed to a registry  
- Kubernetes ultimately relies on the pull action to obtain images  
- Version, registry, authentication, and network all affect whether image delivery succeeds  

Understanding this pipeline clearly will make learning about private registry authentication, Kubernetes image pull policies, Deployment updates, and CI/CD image releases smoother later.  

---

## Twenty-one, Keyword Mnemonics  

- Image naming: Registry address/namespaces/image name:tag  
- Tag: Image version label  
- Build: Build an image  
- Push: Push an image to a registry  
- Pull: Pull an image from a registry  
- Registry: Image registry service  
- imagePullPolicy: Kubernetes image pull policy  
- latest: Ordinary tag, not equal to the absolutely latest available version  

---

## Twenty-two, Operational Extended Understanding  

From an operational perspective, image issues are often not just "build problems," but rather complete delivery pipeline issues.  

Many surface-level Kubernetes deployment anomalies may originate further upstream:  

- Chaotic tag design  
- Poorly configured registry permissions  
- Nodes unable to pull images  
- Tags being repeatedly overwritten  
- Deployment not genuinely switching to the new version  

Therefore, learning image naming, tags, build, push, and pull is not just for "knowing how to operate images," but to accurately locate where the issue occurs in the release, rollback, and troubleshooting processes.  

---

## References  
- Docker Docs: https://docs.docker.com/  
- Docker image tag: https://docs.docker.com/reference/cli/docker/image/tag/  
- Docker push: https://docs.docker.com/reference/cli/docker/image/push/  
- Docker pull: https://docs.docker.com/reference/cli/docker/image/pull/  
- Kubernetes Images: https://kubernetes.io/docs/concepts/containers/images/  

---

## Next Day Recommendation  
Next post recommendation:  

[[04-Container Runtime Basics - Startup Exit Ports Environment Variables Log Viewing]]