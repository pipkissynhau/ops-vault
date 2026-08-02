# 03 - Basics of Image Naming, Tagging, Building, Pushing, and Pulling

## Documentation Notes
- Documentation Focus: Basics of the image lifecycle
- Applicable Phase: Article 4 of Business Containerization Learning / Basics of Image Management and Distribution
- Recommended Path: `04-Kubernetes/07-Application Deployment/01-Image and Container Basics/03- Basics of Image Naming, Tagging, Building, Pushing, and Pulling`

## Tags
#Kubernetes #Docker #Image #Image Repository #tag #Image Building #Image Pushing #Image Pulling #Business Containerization #Cloud Native #Ops #Interview Notes

---

## I. Why Learn About Image Naming, Tagging, Building, Pushing, and Pulling

In the process of business containerization, Dockerfile addresses:

**How to create an image.**

However, image naming, tagging, building, pushing, and pulling deal with:

**How images can be identified, managed, distributed, and used.**

If these aspects are not clear, the following problems may easily occur during subsequent Kubernetes deployment:

- The image name is correct, but the node cannot pull it.
- If the tag is set to `latest`, the wrong version may be deployed.
- Even though the image has been rebuilt, the Pod still does not get updated.
- Different naming conventions are used in development, testing, and production environments, making version tracking difficult.
- Private repository authentication fails, resulting in a `ImagePullBackOff` state for Pods.
- The same image is repeatedly built and pushed by multiple people, leading to version confusion.

Therefore, this section is a crucial step from “being able to write Dockerfiles” to “being capable of managing the image delivery process”.

---

## II. The Basic Chain of the Image Lifecycle

From business programs to Kubernetes operation, an image typically goes through the following processes:

### 1. Write a Dockerfile
Define how the image should be built.

### 2. Build the local image
Use the Dockerfile and business files to build a local image.

### 3. Name and tag the image
Give the image a clear identity and version.

### 4. Push it to the image repository
Enable other machines and Kubernetes nodes to pull it.

### 5. Kubernetes nodes pull the image
Retrieve the image from the repository based on configurations such as Deployment or StatefulSet.

### 6. Start the container
The container runtime starts the business process.

This chain can be summarized as:

**Business files → Dockerfile → Build image → Name and tag → Push to repository → Nodes pull → Start container**

---

## III. The Basic Structure of Image Names

A complete image name usually consists of the following parts:

    Repository address/Namespace/Image name:tag

For example:

    harbor.example.com/project/nginx-web:v1.0.0

This can be understood in four parts.

### 1. Repository address
For example:

- `docker.io`
- `harbor.example.com`
- `registry.example.com`

Indicates which repository service the image is stored in.

### 2. Namespace or project name
For example:

- `library`
- `project`
- `devops`

Used for classification, isolation, and permission management.

### 3. Image name
For example:

- `nginx-web`
- `user-service`
- `nacos-server`

Indicates the specific business image name.

### 4. Tag
For example:

- `v1.0.0`
- `20260401`
- `dev-001`
- `latest`

Indicates the image version or identifier.

---

## IV. What is a Tag

A tag can be understood as:

**A version label for an image.**

Its purpose is not to modify the image content but to assign a recognizable version name to a specific image.

For example:

    app:v1.0.0
    app:v1.0.1
    app:dev
    app:latest

Here, `v1.0.0`, `v1.0.1`, `dev`, and `latest` are all tags.

---

## V. Why Tags Are Important

### 1. Used to distinguish versions
Different tags usually correspond to different version images.

### 2. Used for deployment and rollback
For example:

- The production environment currently uses `v1.0.0`.
- Upgrade to `v1.0.1`.
- Roll back to `v1.0.0` when issues occur.

### 3. Used for environment differentiation
For example:

- `dev`
- `test`
- `prod`

### 4. Used for pipeline management in CI/CD
CI/CD systems often generate tags automatically based on:

- Git commit
- Branch name
- Build number
- Release time

---

## VI. Common Naming Methods for Tags

There are several#### 3. `Never`
Never pull images; use only local mirrors.  
This approach is generally used in special testing scenarios.

---

## Fifteen, Common Issues in Image Management

### 1. Confusion with Tags
Manifestations:

- Uncertainty about which version is online
- Mixing of images across environments
- Difficulties in rolling back

### 2. Using `latest` Makes Version Tracking Impossible
Manifestations:

- Inability to identify the actual build version
- Challenges in troubleshooting
- Unpredictable update behavior

### 3. Images Have Been Rebuilt, But Pods Are Not Updated
Possible Reasons:

- The tag has not changed
- The `imagePullPolicy` is inappropriate
- The Deployment has not triggered rolling updates

### 4. Nodes Cannot Pull Images
Possible Reasons:

- Incorrect repository address
- DNS or network issues
- Repository authentication failure
- Tag does not exist
- Abnormal repository certificate

### 5. Excessive Build Context
Manifestations:

- Slow build process
- Unrelated files being included in the image
- Increased security risks

---

## Sixteen, More Rational Image Management Practices in Production

### 1. Stable Image Names and Clear Tags
For example:

- Keep the image name consistent: `user-service`
- Use tags that reflect version changes: `v1.0.0`

### 2. Unified Naming Standards for Development, Testing, and Production
Avoid using completely different naming conventions for each environment.

### 3. Make Tags Traceable
It is recommended to trace at least:

- Build time
- Build number
- Git commit
- Released version

### 4. Avoid Long-Term Dependence on `latest` in Production
Clear versioning facilitates release, auditing, rollback, and troubleshooting.

### 5. Try to Consolidate Private Business Images in a Corporate Repository
This facilitates permission control and centralized management.

---

## Seventeen, How Operations Understand “Images Not Being Updated”

This is a common issue in production.

### Scenario 1: Releasing the Same Tag Again
For example, still releasing:

    app:latest

But if the Deployment configuration remains unchanged, nodes may not refresh as expected.

### Scenario 2: Local and Repository Images Are Different Versions
The local image has been rebuilt but not pushed; or it has been pushed, but the Pod is still using the old cache.

### Scenario 3: The Deployment Does Not Trigger Updates
Kubernetes often determines whether to perform rolling updates based on whether the Pod Template has changed.

### Key Points for Operations
Do not just focus on “building an image”; consider the entire process:

- Whether the tag was correctly applied
- Whether it was properly pushed
- Whether the Deployment references the new tag
- Whether nodes have actually pulled the new image
- Whether the Pod has been successfully rebuilt

---

## Eighteen, What Level of Understanding Is Required for This Section

At this stage, achieving the following levels is sufficient:

### 1. Be able to explain the basic structure of image names
### 2. Understand the role of tags
### 3. Explain what build, push, and pull mean respectively
### 4. Comprehend how Kubernetes nodes pull images
### 5. Understand why it is not recommended to rely on `latest` in production for long periods
### 6. Be able to preliminarily identify where issues in the image distribution chain occur

---

## Nineteen, Common Follow-up Questions in Interviews

Common questions include:

- What is the basic structure of an image name?
- What is a tag and what is its purpose?
- Why is it not advisable to use `latest` for long periods?
- What do build, push, and pull mean?
- How does Kubernetes pull images?
- What are the different types of `imagePullPolicy`?
- If the image has been updated, why is the Pod still using the old version?
- How to troubleshoot common issues when pulling from a private repository?
- How are image tags typically designed in production?

---

## Twenty, Phase Summary

Image naming, tagging, building, pushing, and pulling are critical steps in transforming an image from a “local build product” into a “standard deliverable usable by the cluster.”

The most important thing here is not to memorize many commands but to establish a clear understanding of the entire image distribution chain:

- Images must have clear names
- Tags must be traceable
- Local building is just the first step
- Only after pushing to the repository can other machines and nodes use them
- Kubernetes ultimately relies on pull operations to obtain images
- Versioning, repository, authentication, and network all affect the success of image delivery

Once you understand this chain clearly, subsequent learning about private repository authentication, Kubernetes image pull strategies, Deployment updates, and CI/CD image releases will be much smoother.

---

## Twenty-One, Key Terms for Quick Reference

- Image Naming: Repository address/namespace