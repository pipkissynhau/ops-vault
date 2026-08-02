# 02-Harbor A-B Image Synchronization: Replication Strategy and Image Repository Disaster Recovery

Recommended path: 08-CI-CD/02-CI-CD Platform Backup and Disaster Recovery/02-Harbor A-B Image Synchronization: Replication Strategy and Image Repository Disaster Recovery.md

Tags: #CI-CD #Harbor #MirrorRepository #Replication #MirrorSync #DisasterPreparedness #MirrorContainer #Kubernetes #SRE

---

## I. Document Overview

Harbor is a commonly used container image repository in enterprise environments, typically used to store container image artifacts built by CI/CD pipelines.

In a typical Kubernetes application delivery pipeline, Harbor resides between Jenkins/GitLab CI and Kubernetes clusters:

    GitLab Code Repository
       |
       v
    Jenkins / GitLab CI
       |
       v
    Build Container Images
       |
       v
    Push to Harbor
       |
       v
    Kubernetes Pulls Images from Harbor
       |
       v
    Pod Launches Application Containers

If Harbor fails, the impact is not limited to "unable to push images", it may also affect:

- New versions cannot be released
- Pods cannot pull images after recreation
- Deployment scaling fails
- Business cannot recover after node reconstruction
- Historical images cannot be found during rollback
- Disaster recovery cluster lacks corresponding images
- CI/CD release pipeline is interrupted

Therefore, in production environments, Harbor image repositories need to consider synchronization, backup, recovery, and disaster recovery capabilities.

This document uses Harbor A and Harbor B as examples to explain the basic principles of Harbor Replication replication strategy, Web UI configuration methods, verification approaches, common troubleshooting, and production considerations.

---

## II. Why Harbor A/B Image Synchronization is Needed

### 2.1 Risks of a Single Harbor

If an enterprise only has one Harbor, common risks include:

- Harbor server failure
- Harbor storage damage
- Harbor database anomalies
- Harbor upgrade failure
- Harbor certificate anomalies
- Harbor network unreachable
- Harbor data center failure
- Harbor project or image mistakenly deleted
- Harbor GC or Retention policy mistakenly deletes critical images
- Harbor permission configuration errors causing image inaccessibility

For Kubernetes, image repository unavailability does not necessarily immediately affect already running Pods, but it affects scenarios requiring image re-pull.

For example:

    1. Pod deleted and rescheduled
    2. Pod drift after node failure
    3. Deployment scaling
    4. Deployment rolling update
    5. New node added to cluster
    6. Local image cache cleared
    7. Business version rollback

If nodes lack local image cache and Harbor is unavailable, it may result in:

    ImagePullBackOff
    ErrImagePull
    Back-off pulling image

---

### 2.2 Value of Harbor A/B Image Synchronization

The core value of Harbor A/B synchronization:

- Image asset redundancy
- Support for disaster recovery restoration
- Support for multi-cluster image pulling
- Support for cross-data center image distribution
- Support for test environment to production environment image flow
- Support for offline environment pre-synchronization of images
- Reduce single-point Harbor failure impact
- Improve image release pipeline reliability

Typical structure:

    CI/CD Pipeline
        |
        v
    Harbor A Primary Image Repository
        |
        | Replication
        v
    Harbor B Backup Image Repository
        |
        v
    Disaster Recovery Cluster / Backup Pull Source

---

## III. What is Harbor Replication

Harbor Replication is Harbor's built-in image replication capability.

It is not simply rsyncing Harbor data directories at the OS level, nor directly copying Harbor database directories, but at the image repository level, synchronizing images, Tags, Artifacts, etc., from one repository to another through Harbor's own replication rules.

Replication can be used for:

- Harbor to Harbor
- Harbor to non-Harbor Registry
- Non-Harbor Registry to Harbor
- Local Harbor to cloud vendor image repository
- Cloud vendor image repository to local Harbor
- Cross-data center, cross-environment, cross-cluster image distribution

Common usage scenarios:

    Harbor A  ----Push-based Replication---->  Harbor B

Or:

    Harbor B  ----Pull-based Replication---->  Harbor A

In enterprise internal A/B image synchronization, the most common scenario is:

    Harbor A primary repository configured with Push-based Replication, pushing images to Harbor B.

---

## IV. Difference Between Push-based and Pull-based

### 4.1 Push-based Replication

Push-based means:

    Current Harbor pushes local images to the remote Registry.

Example:

    Harbor A as source
    Harbor B as target

Process:

    Developer submits code
       |
       v
    Jenkins builds image
       |
       v
    Pushes image to Harbor A
       |
       v
    Harbor A follows replication rules
       |
       v
    Automatically pushes image to Harbor B

Applicable scenarios:

- Harbor A is the primary repository
- CI/CD only pushes to Harbor A
- Harbor B is the backup repository
- Want A to automatically synchronize new images to B
- A can access B's Harbor Web/API/Registry address

---

### 4.2 Pull-based Replication

Pull-based means:

    Current Harbor pulls images from the remote Registry to local.

Example:

    Harbor B as target
    Harbor A as remote source

Process:

    Image already exists in Harbor A
       |
       v
    Harbor B pulls image from Harbor A periodically or manually
       |
       v
    Image saved to Harbor B

Applicable scenarios:

- Target end prefers active pull
- Source end is inconvenient for active push
- Network policy only allows B to access A
- B end prefers scheduled batch synchronization
- Disaster recovery end does not want to expose write endpoints to source end

---

### 4.3 How to choose in production

Common choices:

| Scenario | Recommended Method |
|---|---|
| A is the main repository, B is the backup repository | Push-based |
| B cannot be accessed by A, but B can access A | Pull-based |
| A/B are both in intranet, A to B network is reachable | Push-based |
| Cross-data center, active pull from target end is safer | Pull-based |
| Need immediate synchronization after new image push | Push-based + Event Based |
| Need nightly batch synchronization | Scheduled |

Common design for SMEs:

    Main Harbor configured with Push-based Replication.
    CI/CD only pushes to Harbor A.
    Harbor A triggers synchronization to Harbor B after image push or Retag.
    Harbor B serves as a backup repository normally, can also be used for disaster recovery clusters.

---

## FiveI don't know.Harbor A/B Image Synchronization Architecture

### 5.1 Basic Topology

    ┌──────────────────────┐
    │      GitLab Code Repository  │
    └──────────┬───────────┘
               │
               v
    ┌──────────────────────┐
    │ Jenkins / GitLab CI   │
    └──────────┬───────────┘
               │ docker build
               v
    ┌──────────────────────┐
    │   Harbor A Main Repository     │
    │ harbor-a.example.com  │
    └──────────┬───────────┘
               │ Replication
               v
    ┌──────────────────────┐
    │   Harbor B Backup Repository     │
    │ harbor-b.example.com  │
    └──────────────────────┘

---

### 5.2 Experimental Environment Example

This article assumes:

| Role | Address | Purpose |
|---|---|---|
| Harbor A | harbor-a.example.com | Main image repository |
| Harbor B | harbor-b.example.com | Backup image repository |
| Project | devops | Example project |
| Image | devops/demo-app | Example image |
| Tag | v1.0.0 | Example tag |

If using an experimental environment, IP and port can also be used:

| Role | Address |
|---|---|
| Harbor A | 10.0.0.10:8090 |
| Harbor B | 10.0.0.11:8090 |

Production environment recommends using HTTPS domain:

    https://harbor-a.example.com
    https://harbor-b.example.com

If using HTTP in experimental environment, need to additionally configure insecure registry or registry hosts in Docker, containerd, Kubernetes nodes.

---

## SixI don't know.Preparation Before Configuration

### 6.1 Network Connectivity Check

From Harbor A node to access Harbor B:

    curl -k https://harbor-b.example.com/api/v2.0/systeminfo

If HTTP experimental environment:

    curl http://10.0.0.11:8090/api/v2.0/systeminfo

Check DNS:

    nslookup harbor-b.example.com

Check port:

    nc -vz harbor-b.example.com 443

Or:

    telnet harbor-b.example.com 443

Need to ensure:

- Harbor A can access Harbor B
- Harbor B Web/API is normal
- Harbor B Registry service is normal
- Firewall allows 443 or actual Harbor port
- If using HTTPS, certificate chain is trusted by Harbor A

---

### 6.2 Project Preparation

Recommend creating same-named Projects in both Harbor A and Harbor B, for example:

    devops

Create project in Harbor B:

    Projects -> New Project -> devops

In production environment, recommend pre-planning project names, access permissions, and synchronization rules. Do not recommend universal synchronization for all images.

---

### 6.3 Account Preparation

Not recommended to directly use admin account for replication.

Recommend creating dedicated Robot Account in Harbor B, for example:

    robot$replication-to-b

Permissions recommended:

    Project: devops
    Permission: Pull Repository
    Permission: Push Repository

If using Harbor system-level replication, can also create system Robot Account according to actual situation.

Principle:

    Only grant minimal permissions needed for Harbor A to synchronize to Harbor B.

Do not grant unnecessary project management permissions, deletion permissions, or system administrator permissions.

---

### 6.4 Certificate Preparation

If Harbor B uses enterprise internal CA signed certificate, need to ensure Harbor A trusts this CA.

Otherwise, creating Replication Endpoint may encounter:

    x509: certificate signed by unknown authority

Solutions:

1. Use public trusted certificate
2. Add internal CA to Harbor A system or container trust chain
3. Temporarily disable Verify Remote Cert, not recommended for long-term use in production

Production recommendation:

    Client access to Harbor must use HTTPS.
    Harbor A to Harbor B replication also recommends using HTTPS.
    Even in fully trusted intranet, should at least define network boundaries and access control.

1. Replication Endpoint: Tells Harbor A where Harbor B is located and which account to use for access.
2. Replication Rule: Tells Harbor A which images to synchronize, where to synchronize them, and when to trigger synchronization.

Therefore, the Web UI needs to be enabled with:

    Replication Rule + Event Based trigger mode

After CI/CD pushes images to Harbor A, Harbor A will automatically synchronize them to Harbor B according to the replication rules.

---

### 7.1 Prerequisites

Before configuring the Web UI, the following preparations are needed:

1. Harbor A can access Harbor B.
2. Harbor B has already created the target Project, for example, devops.
3. A Robot Account for synchronization has been created on Harbor B.
4. The Robot Account must at least have Push / Pull permissions for the target Project.
5. If Harbor B uses HTTPS, the certificate must be trusted by Harbor A.

It is not recommended to use the admin account for replication authentication.

Recommended use:

    robot$replication-to-b

---

### 7.2 Creating the Target Project on Harbor B

Log in to Harbor B:

    https://harbor-b.example.com

Navigate to:

    Projects -> New Project

Fill in:

| Configuration Item | Example Value |
|---|---|
| Project Name | devops |
| Access Level | Private |
| Proxy Cache | Disable, regular project is sufficient |

Notes:

    In A/B synchronization scenarios, the target Project on Harbor B is generally created in advance.
    The project name is recommended to be consistent with Harbor A to reduce path conversion and image switching costs.

---

### 7.3 Creating a Robot Account on Harbor B

Navigate to Harbor B:

    Projects -> devops -> Robot Accounts -> New Robot Account

Example configuration:

| Configuration Item | Example Value |
|---|---|
| Name | replication-to-b |
| Description | Used by Harbor A replication |
| Expiration Time | Set according to enterprise standards |
| Permissions | Repository Pull / Repository Push |

Save the Token after creation.

Notes:

    The Robot Account Token will only be fully displayed once, requiring proper storage.
    Do not write the Token into public repositories or ordinary documents.
    In production environments, the Token should be rotated regularly.

---

### 7.4 Creating a Replication Endpoint on Harbor A

Log in to Harbor A Web UI:

    https://harbor-a.example.com

Navigate to:

    Administration -> Registries -> New Endpoint

Fill in the example:

| Configuration Item | Example Value | Notes |
|---|---|---|
| Provider | Harbor | Target repository type |
| Name | harbor-b | Endpoint name |
| Description | Harbor B backup registry | Description |
| Endpoint URL | https://harbor-b.example.com | Harbor B address |
| Access ID | robot$replication-to-b | Robot Account on Harbor B |
| Access Secret | Robot Account Token | Robot Token |
| Verify Remote Cert | Check | Production recommendation: enable |

If it's an HTTP test environment, for example:

    http://10.0.0.11:8090

Then the Endpoint URL should be written as the HTTP address.

If HTTPS uses a self-signed certificate or internal CA, certificate validation may fail. In production environments, it's recommended to correctly add the internal CA to the trust chain. Long-term disabling certificate validation is not advised.

After filling in, click:

    Test Connection

Test successfully, then click:

    OK

---

### 7.5 Creating a Replication Rule on Harbor A

Navigate to Harbor A Web UI:

    Administration -> Replications -> New Replication Rule

Fill in the example:

| Configuration Item | Recommended Value | Notes |
|---|---|---|
| Name | push-devops-to-harbor-b | Replication rule name |
| Description | Push devops images to Harbor B | Description |
| Replication Mode | Push-based | Push from Harbor A to Harbor B |
| Destination Registry | harbor-b | Select the previously created Endpoint |
| Resource Type | Artifact / Image | Depends on Harbor version |
| Source Resource Filter - Name | devops/** | Synchronize all images under devops project |
| Source Resource Filter - Tag | v* / release-* / * | Filter by Tag |
| Destination Namespace | devops | Target Project |
| Trigger Mode | Event Based | Automatically trigger after image push |
| Bandwidth | -1 | No bandwidth limit |
| Override | Caution: enable | Whether to overwrite same-named Tags on target side |
| Delete remote resources when locally deleted | Not recommended: enable | Avoid accidental deletion spread in disaster recovery scenarios |

Production recommendations:

    Replication Mode: Push-based
    Trigger Mode: Event Based
    Delete remote resources when locally deleted: Disable

---

### 7.6 How to Choose Trigger Mode

Harbor Replication has three common trigger methods:

| Trigger Method | Meaning | Use Cases |
|---|---|---|
| Manual | Manually click to execute synchronization | Initial synchronization, catch-up synchronization |
| Scheduled | Timed synchronization | Nighttime batch synchronization, low-traffic cross-datacenter synchronization |
| Event Based | Event-triggered synchronization | Automatically synchronize after mirror Push |

If the requirement is:

    Push to Harbor A, then Harbor B automatically synchronizes

Then select:

    Event Based

This is the most critical "auto-synchronization switch" on the Web UI.

---

### 7.7 How to Write Name Filter

Name Filter is used to control which repositories to synchronize.

Common examples:

| Rule | Description |
|---|---|
| devops/** | Synchronize all images under the devops project |
| devops/demo-app | Synchronize only devops/demo-app |
| middleware/** | Synchronize all images under the middleware project |
| ** | Synchronize all projects, production use with caution |

If the image address is:

    harbor-a.example.com/devops/demo-app:v1.0.0

You can configure:

    devops/demo-app

Or synchronize the entire project:

    devops/**

---

### 7.8 How to Write Tag Filter

Tag Filter is used to control which image versions to synchronize.

Common examples:

| Rule | Description |
|---|---|
| v* | Synchronize versions starting with v |
| release-* | Synchronize versions starting with release |
| prod-* | Synchronize versions starting with prod |
| * | Synchronize all Tags |
| latest | Synchronize only latest, not recommended for production |

Production recommendations: synchronize trackable versions, e.g.:

    v1.0.0
    release-20260428-001
    main-a1b2c3d-1024

Not recommended to rely only on:

    latest

---

### 7.9 Should Override Be Enabled

Override means:

    When Harbor B already has an image with the same name and Tag, whether to overwrite it.

Production recommendation:

    Try to disable Override.
    Use immutable Tags to manage versions.

Reasons:

    Repeated overwriting of the same Tag makes it difficult to track content between A/B repositories.
    AlsoNot good. rollback and audit.

In test environments, if you frequently reuse latest, you can enable it based on needs.

---

### 7.10 Should Delete Synchronization Be Enabled

Delete remote resources when locally deleted means:

    When Harbor A deletes an image, Harbor B also synchronizes the deletion.

Not recommended in disaster recovery scenarios.

Reasons:

    If Harbor A mistakenly deletes production images, Harbor B will also be deleted.
    This would make the disaster recovery repository lose its protective value.

Recommended strategy:

    Synchronize new and updated content.
    Do not synchronize deletion actions to the disaster recovery repository.

---

### 7.11 Manually Execute a Synchronization

After creating the rule, you can manually execute it once to verify the configuration.

Navigate to:

    Administration -> Replications

Select the rule you just created:

    push-devops-to-harbor-b

Click:

    Replicate

Then check:

    Executions
    Task Logs
    Status

If the status is:

    Succeed

It means the rule can execute normally.

If it fails, prioritize checking Task Logs.

---

### 7.12 Verify Auto-Synchronization by Pushing an Image

Push a test image to Harbor A from the client:

    docker login harbor-a.example.com

    docker pull nginx:1.25
    docker tag nginx:1.25 harbor-a.example.com/devops/nginx:v1.0.0
    docker push harbor-a.example.com/devops/nginx:v1.0.0

Then check in Harbor A Web UI:

    Administration -> Replications -> Executions

Confirm if a replication task is automatically generated.

Log in to Harbor B and check:

    Projects -> devops -> Repositories

Confirm if:

    nginx:v1.0.0

Appears.

Finally pull from Harbor B to verify:

    docker login harbor-b.example.com
    docker pull harbor-b.example.com/devops/nginx:v1.0.0

---

### 7.13 Web UI Configuration Summary

Harbor A/B auto-synchronization requires three steps in the Web UI:

    Step 1: Create the target Project and Robot Account in Harbor B.
    Step 2: Create a replication endpoint pointing to Harbor B in Harbor A.
    Step 3: Create a push-based replication rule in Harbor A, setting Trigger Mode to Event Based.

Core configuration paths:

    Administration -> Registries -> New Endpoint
    Administration -> Replications -> New Replication Rule
    Replication Mode -> Push-based
    Trigger Mode -> Event Based

Production notes:

    1. Use Robot Account, not admin.
    2. Use HTTPS, do not disable certificate verification long-term.
    3. Do not synchronize deletion actions.
    4. Do not synchronize only latest.
    5. Do not arbitrarily enable Override.
    6. After successful synchronization, must actually docker pull from Harbor B to verify.

---

## EightI don't know.Replication Rule Key Configuration Explanation

### 8.1 Replication Mode

Common choice:

    Push-based

Meaning:

    Harbor A pushes matching rule images to Harbor B.

Suitable for master-slave synchronization:

    Harbor A -> Harbor B

---

### 8.2 Destination Registry

Select the endpoint created earlier:

    harbor-b

This Endpoint contains:

- Access address of Harbor B
- Access account for Harbor B
- Access Token for Harbor B
- Certificate validation method

---

### 8.3 Source Resource Filter - Name

Name Filter is used to control which image repositories are synchronized.

Example:

    devops/**
    middleware/**
    devops/demo-app

If the image is:

    harbor-a.example.com/devops/demo-app:v1.0.0

You can write:

    devops/demo-app

If you want to synchronize all images under the devops project, you can write:

    devops/**

It is not recommended to directly write:

    **

In production environments unless you explicitly want to synchronize all resources from Harbor A.

---

### 8.4 Source Resource Filter - Tag

Tag Filter is used to control which image versions are synchronized.

Example:

    v*
    release-*
    prod-*
    *

Production recommendation:

    Synchronize explicitly traceable version Tags.
    Do not synchronize only latest.

Recommended Tag examples:

    v1.0.0
    release-20260428-001
    main-a1b2c3d-1024
    dev-7f3a9c2-356

---

### 8.5 Destination Namespace

Destination Namespace indicates which project or namespace in Harbor B the synchronization is targeted to.

Example:

    devops

Source image:

    harbor-a.example.com/devops/demo-app:v1.0.0

Target image:

    harbor-b.example.com/devops/demo-app:v1.0.0

If the Destination Namespace configuration is incorrect, it may cause synchronization success but the image to appear in other projects in Harbor B.

---

### 8.6 Destination Flattening

Flattening is used to control whether the target path hierarchy is flattened.

In ordinary Harbor A/B same-project synchronization scenarios, complex configurations are typically not needed.

Recommendation:

    Keep the default non-flattened or plan according to actual path.
    After configuration, must verify the result through the image path on Harbor B.

---

### 8.7 Bandwidth

Bandwidth is used to limit the synchronization bandwidth.

Common configuration:

    -1

Indicates no limit.

When cross-datacenter or bandwidth is limited, you can restrict it according to actual conditions to avoid replication tasks affecting business traffic.

---

## NineI don't know.Synchronization Verification

### 9.1 Push Test Image to Harbor A

Login to Harbor A:

    docker login harbor-a.example.com

If it's an HTTP experimental environment, Docker or containerd needs to be configured to trust insecure registry.

Pull test image:

    docker pull nginx:1.25

Re-tag:

    docker tag nginx:1.25 harbor-a.example.com/devops/nginx:v1.0.0

Push to Harbor A:

    docker push harbor-a.example.com/devops/nginx:v1.0.0

---

### 9.2 View Harbor A Replication Tasks

Enter Harbor A:

    Administration -> Replications

Click the corresponding rule to view execution records.

Focus on:

- Execution ID
- Status
- In Progress
- Succeed
- Failed
- Stopped
- Task Logs

If failed, click the log icon to view specific errors.

---

### 9.3 Check if Image Appears in Harbor B

Enter Harbor B:

    Projects -> devops -> Repositories

You should see:

    devops/nginx

Tag:

    v1.0.0

---

### 9.4 Pull Image from Harbor B for Verification

Login to Harbor B:

    docker login harbor-b.example.com

Pull image:

    docker pull harbor-b.example.com/devops/nginx:v1.0.0

If it can be pulled normally, it indicates synchronization success.

---

### 9.5 Kubernetes Use Harbor B for Verification

Create test namespace:

    kubectl create namespace harbor-sync-test

Create imagePullSecret:

    kubectl -n harbor-sync-test create secret docker-registry harbor-b-secret \
      --docker-server=harbor-b.example.com \
      --docker-username='robot$k8s-pull' \
      --docker-password='Robot.Token' \
      --docker-email='ops@example.com'

Create test Pod:

    cat > nginx-harbor-b-test.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-harbor-b-test
      namespace: harbor-sync-test
    spec:
      imagePullSecrets:
        - name: harbor-b-secret
      containers:
        - name: nginx
          image: harbor-b.example.com/devops/nginx:v1.0.0
          imagePullPolicy: IfNotPresent
    EOF

Apply:

    kubectl apply -f nginx-harbor-b-test.yaml

Check:

    kubectl -n harbor-sync-test get pod
    kubectl -n harbor-sync-test describe pod nginx-harbor-b-test

Cleanup: /think

kubectl delete -f nginx-harbor-b-test.yaml
kubectl delete namespace harbor-sync-test

---

## 10. Manual Synchronization

If Event Based synchronization fails, or if some images were not synchronized previously, you can manually execute Replication.

Path:

    Administration -> Replications

Select rule:

    push-devops-to-harbor-b

Click:

    Replicate

Then check the execution status and task logs.

Applicable scenarios:

- Synchronize historical images after creating a new rule
- Synchronize after network recovery
- Initial synchronization for Harbor B
- Re-execute after a failed replication
- Verify if the rule is correct

---

## 11. Scheduled Synchronization Strategy

If you don't want images to synchronize immediately after each Push, you can use Scheduled.

Applicable scenarios:

- Cross-regional bandwidth is limited
- Synchronization costs are lower at night
- Disaster recovery repository only needs approximate synchronization
- Large number of images, and you don't want to affect daytime releases
- Test environment to production repository has a defined synchronization window

Example strategy:

| Scenario | Cron |
|---|---|
| Every day at 2:00 AM | 0 2 * * * |
| Every 6 hours | 0 */6 * * * |
| Every Sunday at 3:00 AM | 0 3 * * 0 |

Note:

    Scheduled synchronization reduces real-time performance.
    If RPO requirements are very low, choose Event Based or higher frequency synchronization.

---

## 12. Difference Between A/B Synchronization and Harbor Backup

Harbor Replication is not equal to Harbor full backup.

### 12.1 What Replication Can Solve

Replication mainly solves:

- Image replication
- Artifact synchronization
- Tag synchronization
- Cross-repository distribution
- Primary/secondary repository image redundancy
- Disaster recovery repository pre-retention of image artifacts

---

### 12.2 What Replication Cannot Fully Replace

Replication cannot fully replace:

- Harbor database backup
- Harbor configuration backup
- User permission backup
- Project metadata backup
- Robot Account backup
- Scan report backup
- Audit log backup
- System configuration backup
- Certificate backup
- Storage backend backup

Example:

    Harbor A has project permissions, users, Robot Accounts, scan policies, retention policies, and webhook configurations.
    Replication mainly synchronizes image resources, which does not imply these management configurations are fully synchronized.

Therefore, production disaster recovery should have two layers:

    First layer: Replication ensures image assets exist in Harbor B.
    Second layer: Harbor itself backup ensures platform configuration, database, and metadata can be recovered.

---

## 13. Using Harbor B as a Backup Repository

### 13.1 Do Not Use Harbor B Directly in Normal Operation

Common pattern:

    CI/CD pushes to Harbor A
    Kubernetes production cluster defaults to pulling from Harbor A
    Harbor B serves only as a backup repository

During failure:

    Switch the image address from harbor-a.example.com to harbor-b.example.com

Issues:

    Need to modify Deployment, Helm values, Kustomize, or GitOps repository image addresses.

---

### 13.2 Switch via Unified Domain

Better approach:

    registry.example.com

Normal resolution to Harbor A:

    registry.example.com -> Harbor A

Switch to Harbor B during failure:

    registry.example.com -> Harbor B

Advantages:

- Kubernetes image address remains unchanged
- CI/CD configuration changes are minimal
- Failover is faster
- Operations are more unified

Note:

- Harbor A/B projects, repositories, and Tags must be consistent
- Certificates must cover the unified domain
- DNS TTL should not be too long
- Confirm Harbor B image completeness before switching
- The registry address in imagePullSecret must match the unified domain

---

### 13.3 Kubernetes Multi-Repository Pull Strategy

Kubernetes native Pod image field usually only writes one image address:

    image: registry.example.com/devops/demo-app:v1.0.0

Kubernetes itself will not automatically pull from Harbor A and switch to Harbor B if it fails.

To achieve better image pull disaster recovery, consider:

- Unified domain switching
- Node-level image caching
- Local Registry Mirror
- Harbor Proxy Cache
- Switch image address at CI/CD or GitOps layer
- Parameterize registry address in Helm values

Helm values example:

    image:
      registry: registry.example.com
      repository: devops/demo-app
      tag: v1.0.0

During failure, only adjust:

    image.registry=harbor-b.example.com

---

## 14. Common Issues and Troubleshooting

### 14.1 Endpoint Test Connection Failed

Possible causes:

- Harbor B address is incorrect
- Harbor B service is not running
- Network is unreachable
- Firewall is not allowing traffic
- DNS resolution is abnormal
- Certificate is not trusted
- Username/password is incorrect

Troubleshoot:

    curl -k https://harbor-b.example.com
    curl -k https://harbor-b.example.com/api/v2.0/systeminfo
    nslookup harbor-b.example.com
    nc -vz harbor-b.example.com 443

Check Harbor B service:

    docker compose ps

Or:

    kubectl get pod -n harbor

---

### 14.2 Replication Failed: unauthorized

Possible causes:

- Endpoint Access ID is incorrect
- Access Secret is incorrect
- Robot Account Token expired or copied incorrectly
- Harbor B user lacks permissions
- Target project permissions are insufficient

Resolution:

    1. Regenerate Robot Account Token
    2. Confirm Robot Account has Push permissions for the target project
    3. Reconfigure Endpoint
    4. Click Test Connection
    5. Manually execute Replicate

---

### 14.3 Replication Execution Failed: forbidden

Possible Causes:

- Robot Account has only Pull, no Push
- Insufficient permissions for target project
- Target project does not allow current account to push
- Account scope does not include target Project

Resolution:

    1. Check Harbor B's Robot Account permissions
    2. Confirm Repository Push permissions exist
    3. Confirm target Project is devops
    4. Regenerate Token and update Endpoint

---

### 14.4 Replication Execution Failed: project not found

Possible Causes:

- Harbor B does not have corresponding Project
- Destination Namespace configuration is incorrect
- Source project and target project names are inconsistent
- Flattening causes path mismatch

Resolution:

    1. Create target Project in Harbor B
    2. Check Destination Namespace
    3. Check Name Filter
    4. Check Destination Flattening
    5. Re-execute synchronization

---

### 14.5 Synchronization Succeeded but Harbor B Cannot Find Image

Possible Causes:

- Synchronized to different Namespace
- Flattening changed image path
- Tag filtering rules do not match
- Resource filtering only synchronized partial types
- Checked wrong project or repository

Troubleshooting:

    1. Check Replication task logs
    2. Check target Namespace
    3. Search for image name in Harbor B
    4. Check Destination Flattening settings
    5. Manually docker pull to verify

---

### 14.6 Event Based Does Not Trigger Automatically

Possible Causes:

- Trigger Mode is not Event Based
- Pushed image does not match filtering rules
- Tag mismatch
- Image pushed to other Project
- Rule is disabled
- JobService abnormal
- Harbor A internal task queue abnormal

Troubleshooting:

    1. Check if Replication Rule is enabled
    2. Check Name Filter and Tag Filter
    3. Check JobService status
    4. View Replication Execution
    5. Check Harbor logs

Docker Compose deployment log check:

    docker compose logs -f jobservice
    docker compose logs -f core

Kubernetes deployment log check:

    kubectl logs -n harbor deploy/harbor-jobservice
    kubectl logs -n harbor deploy/harbor-core

---

### 14.7 Image Synchronization is Very Slow

Possible Causes:

- Image layers are large
- Cross-datacenter bandwidth is insufficient
- Synchronization concurrency is too high
- Target Harbor storage is slow
- Network jitter
- Harbor JobService resources are insufficient
- Registry backend storage performance is inadequate

Optimization Directions:

    1. Reduce image size
    2. Avoid redundant large base layer builds
    3. Set reasonable synchronization time window
    4. Use Scheduled to avoid peak hours
    5. Adjust bandwidth limits
    6. Optimize Harbor backend storage
    7. Increase JobService resources
    8. Preheat base images in advance

---

### 14.8 Harbor B Pull Image Failed

Possible Causes:

- Image was not synchronized successfully
- imagePullSecret configuration is incorrect
- Kubernetes nodes do not trust Harbor B certificate
- Using HTTP registry but containerd/Docker is not configured with insecure registry
- Image address is written incorrectly
- Harbor B has insufficient permissions

Troubleshooting:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Test on node:

    crictl pull harbor-b.example.com/devops/nginx:v1.0.0

Or:

    ctr -n k8s.io images pull harbor-b.example.com/devops/nginx:v1.0.0

---

### 14.9 HTTP Harbor Pull Failed in Kubernetes

If experimental environment Harbor uses HTTP, for example:

    10.0.0.10:8090

Docker login success does not guarantee Kubernetes nodes can pull.

Reasons:

    Kubernetes pulls images via container runtime.
    If runtime is containerd, need to configure containerd's registry hosts.
    Docker's daemon.json configuration may not affect containerd.

containerd example path:

    /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml

Example content:

    server = "http://10.0.0.10:8090"

    [host."http://10.0.0.10:8090"]
      capabilities = ["pull", "resolve", "push"]
      skip_verify = true

Restart containerd:

    systemctl restart containerd

Verification:

    crictl pull 10.0.0.10:8090/devops/nginx:v1.0.0

Production Recommendations:

    Clients accessing Harbor via public network or business should use HTTPS.
    HTTP is more suitable for temporary experimental environments or clearly trusted internal network test environments.

---

## FifteenI don't know.Production Design Recommendations

### 15.1 Do Not Only Synchronize latest

Not recommended:

    harbor-a.example.com/devops/demo-app:latest

Recommended: /think

harbor-a.example.com/devops/demo-app:v1.0.0  
harbor-a.example.com/devops/demo-app:main-a1b2c3d-1024  
harbor-a.example.com/devops/demo-app:release-20260428-001  

Reasons:  

- `latest` cannot accurately trace back  
- `latest` is easily overwritten  
- `latest` is not conducive to rollback  
- `latest` is not conducive to determining if Harbor A/B are consistent  

---

### 15.2 Production Image Tag Should Be Immutable  

Recommendations:  

    A Tag should correspond to a single build artifact.  
    Do not repeatedly overwrite the same production Tag.  

Benefits:  

- Traceable  
- Rollback-capable  
- Auditable  
- Facilitates comparison between A/B repositories  
- Avoids confusion caused by synchronized overwrites  

---

### 15.3 Harbor B Should Not Arbitrarily Delete Images  

The value of a disaster recovery repository lies in retaining recoverable images.  

Recommendations:  

- Harbor B's retention period can be longer than Harbor A  
- Enable synchronization deletion cautiously  
- Confirm retention policies before GC  
- Add protection tags to critical version images  
- Do not easily clean up released versions  

---

### 15.4 Use Minimal Permissions for Replication Accounts  

Not recommended:  

    Use admin account to configure Endpoint  

Recommendations:  

    Use Robot Account  

Permission controls:  

- Only allow access to target Project  
- Grant only Push/Pull permissions needed  
- Rotate Token regularly  
- Reclaim accounts after employee departure or project decommissioning  
- Do not write Token into public repositories  

---

### 15.5 HTTPS and Certificates  

Production environment recommendations:  

- Harbor provides HTTPS to clients  
- Use HTTPS for synchronization between Harbor A/B  
- Use trusted CA or unified internal CA  
- Avoid long-term disabling certificate validation  
- Replace certificates before they expire  
- Kubernetes nodes must also trust Harbor certificates  

If using internal CA, synchronize to:  

- Harbor A host  
- Harbor B host  
- Docker nodes  
- containerd nodes  
- Kubernetes worker nodes  
- CI/CD build nodes  

---

### 15.6 Clearly Define RPO and RTO  

Harbor A/B synchronization also requires defining RPO and RTO.  

RPO:  

    Recovery Point Objective, recovery point objective.  
    Indicates how far behind Harbor B can be from Harbor A at most.  

If using Event Based:  

    Theoretically near real-time, but still depends on task queue, network, and image size.  

If using Scheduled:  

    RPO depends on the synchronization cycle.  

RTO:  

    Recovery Time Objective, recovery time objective.  
    Indicates how long it takes to switch to Harbor B after Harbor A fails.  

Factors affecting RTO:  

- Whether there is a unified domain name  
- How long DNS TTL is  
- Whether Kubernetes image addresses are hard-coded  
- Whether Helm values parameterize the registry  
- Whether Harbor B has been validated as available  
- Whether imagePullSecret is pre-configured  
- Whether certificates are pre-trusted  

---

## Sixteen, Harbor A/B Synchronization Verification Checklist  

| Check Item | Verification Method | Passed |  
|---|---|---|  
| Harbor A is accessible | Browser or curl |  |  
| Harbor B is accessible | Browser or curl |  |  
| Network connectivity from A to B | curl/nc/telnet |  |  
| Project exists in B | Harbor B project list |  |  
| Robot Account permissions | Harbor B project permissions |  |  
| Endpoint connection successful | Test Connection |  |  
| Replication Rule enabled | Replications page |  |  
| Name Filter correct | Check matching rules |  |  
| Tag Filter correct | Check Tag rules |  |  
| Trigger Mode correct | Event/Scheduled/Manual |  |  
| Test image push to A | docker push |  |  
| Image appears in B | Harbor B project repository |  |  
| B can pull image | docker pull |  |  
| K8s can pull B image | Test Pod |  |  
| Failure logs are viewable | Replication Task Log |  |  
| Fault switch plan is clear | DNS/Helm/GitOps |  |  

---

## Seventeen, Interview Answering Approach  

If asked in an interview:  

    There are two Harbors, Harbor A and Harbor B. After pushing to A, how to automatically synchronize to B?  

You can answer:  

    This is generally achieved using Harbor's built-in Replication strategy, not recommended to directly rsync Harbor's data directory.  
    First, create the target Project and dedicated Robot Account on Harbor B, granting the Robot Account Push/Pull permissions for the target project.  
    Then, in Harbor A's Web UI, go to Administration -> Registries, create a replication endpoint pointing to Harbor B, fill in Harbor B's address, Robot Account, and Token, and test the connection.  
    After the endpoint test is successful, go to Administration -> Replications to create a replication rule.  
    Select Push-based, set Destination Registry to Harbor B, use Name Filter to control which projects or repositories to synchronize, e.g., devops/**, and Tag Filter to control which versions to synchronize, e.g., v* or release-*.  
    If the requirement is to automatically synchronize to Harbor B after pushing to Harbor A, set Trigger Mode to Event Based.  
    This way, CI/CD pushes the image to Harbor A, which automatically triggers the replication task to synchronize matching images to Harbor B.  
    After synchronization, check if the image and Tag exist on Harbor B, and verify if they can be pulled via docker pull or Kubernetes test Pod.  
    In production, also pay attention to certificates, network, firewall, minimal permissions for Robot Account, immutable Tags, cautious enablement of synchronization deletion, and note that Harbor B is not equal to full backup. Replication mainly solves image asset redundancy, while Harbor's database, configuration, certificates, and metadata still need separate backups.

## Eighteen, Relationship with CI/CD Pipeline

### 18.1 Jenkins Build Push to Harbor A

Common pipeline steps:

    git clone
    docker build
    docker login harbor-a
    docker push harbor-a
    kubectl set image or helm upgrade

Example logic:

    IMAGE=harbor-a.example.com/devops/demo-app:${BRANCH_NAME}-${GIT_COMMIT}-${BUILD_NUMBER}

    docker build -t ${IMAGE} .
    docker push ${IMAGE}

After Harbor A receives the image, it automatically synchronizes to Harbor B through Event Based Replication.

---

### 18.2 Use Harbor B During Deployment

Common patterns:

| Mode | Description |
|---|---|
| Use Harbor A normally, switch to Harbor B in case of failure | Simple and common |
| Multi-cluster pull from local Harbor | Suitable for multi-datacenter scenarios |
| Use unified registry domain name | Smoother switching |
| Parameterize registry in GitOps | Suitable for Argo CD/Helm |

Helm values example:

    image:
      registry: harbor-a.example.com
      repository: devops/demo-app
      tag: main-a1b2c3d-1024

Switch to Harbor B:

    image:
      registry: harbor-b.example.com
      repository: devops/demo-app
      tag: main-a1b2c3d-1024

---

## Nineteen, Do Not Misunderstand Replication as High Availability Cluster

Harbor A/B Replication is more akin to:

    Image repository-level data synchronization and disaster recovery

Not equal to:

    Harbor high availability cluster

Differences:

| Type | Description |
|---|---|
| Replication | Mirror images between two Harbor instances |
| HA | Multiple Harbor components provide the same service entry |
| Backup Recovery | Full recovery capability for database, configuration, storage, certificates, etc. |
| Registry Mirror | Image pull caching or proxy |
| Proxy Cache | Proxy remote repository and cache images |

Harbor A/B synchronization can reduce single-point risks, but still requires:

- Harbor itself backup
- Database backup
- Storage backup
- Configuration backup
- Certificate backup
- Recovery drills

---

## Twenty, Production Deployment Recommendations

Recommended approach for small-to-medium enterprises:

    1. Harbor A as the primary repository.
    2. Harbor B as the backup repository.
    3. CI/CD only pushes to Harbor A.
    4. Harbor B pre-creates target Project.
    5. Harbor B creates dedicated Robot Account.
    6. Harbor A configures Replication Endpoint pointing to Harbor B.
    7. Harbor A creates Push-based Replication Rule.
    8. Prefer Event Based Trigger Mode.
    9. Use version number, commit sha, pipeline ID for image Tag.
    10. Disable or cautiously enable deletion synchronization in production.
    11. Harbor B regularly pulls and verifies.
    12. Design switch methods in advance for critical systems.
    13. Still perform database, configuration, and certificate backup for Harbor platform.
    14. Regularly conduct Harbor B pull and disaster recovery switch drills.

---

## Twenty-one, Reference Documents

- Harbor Docs: Creating Replication Endpoints
  https://goharbor.io/docs/2.12.0/administration/configuring-replication/create-replication-endpoints/

- Harbor Docs: Creating a Replication Rule
  https://goharbor.io/docs/2.10.0/administration/configuring-replication/create-replication-rules/

- Harbor Docs: Running Replication Manually
  https://goharbor.io/docs/2.12.0/administration/configuring-replication/manage-replications/

- Harbor Docs: Harbor Compatibility List
  https://goharbor.io/docs/2.4.0/install-config/harbor-compatibility-list/