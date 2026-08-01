# 04-Helm Deployment Practice Guide: Installing, Viewing, Upgrading, and Uninstalling an Application

## Document Notes
- Document Positioning: Helm Basic Practice Cycle Introduction
- Applicable Stage: After understanding Helm basic concepts, common commands, and the role of values.yaml, entering a complete Helm usage process practice
- Recommended Path: `04-Kubernetes/07-Apply deployment/11-Helm and application package management/04-Helm Deployment practice primer: installation, viewing, upgrading and unloading of an application`

## Tags
#Kubernetes #Helm #Chart #Release #values.yaml #install #upgrade #rollback #uninstall #ApplyPackageManagement #DeploymentPractices #Clouds. #Transport

---

## One, What This Document Solves

The previous articles have established a basic understanding of Helm:

- Helm is a Kubernetes application package management tool
- Chart, Release, Repository, Values are core concepts
- Common commands include:
  - `repo`
  - `search`
  - `install`
  - `upgrade`
  - `rollback`
  - `uninstall`
- `values.yaml` is used for parameterized customization of deployment results

At this point, the basic concepts of Helm have been established.  
The focus of this document is to connect these scattered pieces into a complete practice cycle.

This document does not aim to deeply explore a complex Chart, but to establish the following most basic operational path:

- Add repository
- Find Chart
- View default values
- Prepare custom values
- Install application
- View installation status
- Upgrade application
- View change results
- Uninstall application

This path can basically cover the most common basic Helm usage scenarios.

---

## Two, Why Do a Complete Helm Practice

If Helm learning stops at the concept and command level, common results are:

- Know what Helm is
- Know `helm install`
- Know `values.yaml`
- But when actually using it, the operation sequence is unclear

Therefore, after truly entering the usage stage, the most needed is to establish a stable usage process.

A most basic Helm practice usually needs to answer the following questions at least:

- Where does the Chart come from
- How to confirm if the required component exists in the repository
- What do the default parameters look like
- How to write custom values
- How to view the status after Release installation
- How to verify after upgrading
- How to clean up finally

This is also the focus of this document.

---

## Three, Practice Objectives

This document uses a relatively simple and easy-to-understand component for practice.  
The goal is not to delve into the component itself, but to practice the complete Helm process.

The practice objectives of this document are as follows:

- Use Helm to install an nginx application
- Use custom values to control deployment results
- View Release status
- Perform an upgrade
- View upgrade results
- Finally uninstall the Release

The Chart used in this example is:

- `bitnami/nginx`

The reason for choosing nginx is simple:

- Relatively intuitive structure
- Easy to verify
- Doesn't involve complex initialization logic
- More suitable for Helm process entry

---

## Four, Preparations Before Practice

### 1. Confirm Helm is Installed

First confirm the Helm client is available:

    helm version

Normally, you should see Helm version information.

### 2. Confirm Kubernetes Cluster is Available

    kubectl get nodes

Normally, you should see node information.

### 3. Suggested Preparation of a Separate Namespace

To make the practice object clearer, it is recommended to use a separate namespace for this document, for example:

    kubectl create namespace helm-demo

If the namespace already exists, you can continue using it.

---

## Five, Add Helm Repository

### Command

    helm repo add bitnami https://charts.bitnami.com/bitnami

### Explanation

This step means:

- Add a Helm repository named `bitnami`
- You can install Chart from this repository later

### Update Repository Index

    helm repo update

### Explanation

This step means:

- Update the local repository index
- Refresh available Chart and version information

### A Common Practice

In actual use, these two steps are often executed consecutively:

    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update

---

## Six, Search for Available Chart

### Command

    helm search repo nginx

### Explanation

This step is used to confirm:
- Whether the nginx Chart exists in the repository
- Whether the Chart name is correct
- What the current visible version is approximately

If you only want to see the nginx in the bitnami repository, you can also write:

    helm search repo bitnami/nginx

### Operations Understanding Focus

Don't install directly when uncertain, it's a more stable habit to search first.

---

## Seven, View Chart Default Values

### Command

    helm show values bitnami/nginx

### Explanation

The focus of this step is to first understand what parameters this Chart supports, for example:

- Replica count
- Image parameters
- Service type
- Resource limits
- Ingress configuration
- Node scheduling parameters

### Why This Step is Important

When using Helm in practice, you shouldn't guess field names.  
A more reasonable approach is:

- First view the Chart's default values
- Then decide which fields to override

### A Suggestion

If the content is extensive, you can redirect it to a file for easier reading:

    helm show values bitnami/nginx > nginx-default-values.yaml

This allows you to view it slowly in your local editor.

---

## Eight, Prepare Custom Values File

The following provides a teaching-oriented minimal values example.

### File: `values-nginx.yaml`

    replicaCount: 2

    service:
      type: ClusterIP
      ports:
        http: 80

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 300m
        memory: 256Mi

### What This Values File Expresses

#### `replicaCount: 2`
Indicates that nginx expects to run with 2 replicas.

#### `service.type: ClusterIP`
Indicates that the Service uses cluster internal access.

#### `service.ports.http: 80`
Indicates that the exposed HTTP port is 80.

#### `resources`
Sets basic resource requests and limits for the container.

### Why Only a Few Parameters Are Changed Here

Because this article focuses on the Helm usage process, not a complete list of nginx Chart parameters.  
Therefore, only the most commonly used and easily visible effects are modified.

---

## IX. Installing the Release

### Command

    helm install my-nginx bitnami/nginx -n helm-demo -f values-nginx.yaml

If the namespace does not exist, it can also be written as:

    helm install my-nginx bitnami/nginx -n helm-demo --create-namespace -f values-nginx.yaml

### Explanation

This command represents:

- Release name: `my-nginx`
- Chart: `bitnami/nginx`
- Namespace: `helm-demo`
- Parameter source: `values-nginx.yaml`

### What This Step Completes

Helm will render the final Kubernetes objects based on:
- Chart templates
- Default values
- Custom values

And install them into the cluster.

---

## X. What to Check After Installation

### 1. View the Release List

    helm list -n helm-demo

### Explanation

Focus on:
- Whether you can see `my-nginx`
- Whether the status is `deployed`

### 2. View the Release Status

    helm status my-nginx -n helm-demo

### Explanation

Focus on:
- The current status of the Release
- Chart information
- Summary of rendered resources
- NOTES prompts

### 3. View the Current Values

    helm get values my-nginx -n helm-demo

If you want to see a more complete result:

    helm get values my-nginx -n helm-demo -a

### Explanation

This step is used to confirm:
- Which parameters the current Release actually uses
- Whether the custom values take effect

---

## XI. Verify Object Creation from the kubectl Perspective

Helm is an application management entry point, but the objects ultimately reside in the Kubernetes cluster.  
Therefore, after installation, you still need to return to `kubectl` to verify the resource status.

### View Workloads and Service

    kubectl get deploy,svc,pod -n helm-demo

### Expected Phenomenon

You should typically see:

- nginx-related Deployment
- Corresponding Service
- 2 running Pods

### View Pod Details

    kubectl get pod -n helm-demo -o wide

### View Service Details

    kubectl get svc -n helm-demo
    kubectl describe svc -n helm-demo

### Operations Understanding Focus

Helm installation success should not only check `helm list`, but also check:

- Whether objects are genuinely created
- Whether Pods are Running
- Whether Service is available

---

## XII. Perform a Basic Access Verification

If the Service is ClusterIP, the simplest way to verify is usually port forwarding.

### Port Forwarding Command

    kubectl port-forward svc/my-nginx 8080:80 -n helm-demo

If the Service name is not `my-nginx`, you can first confirm the name through:

    kubectl get svc -n helm-demo

### Local Access

Then access locally:

    http://127.0.0.1:8080

### Explanation

If you can see the nginx page, it indicates:

- The Release installation was successful
- Pods are running normally
- Service is forwarding correctly
- The basic Helm practice loop is established

---

## XIII. Execute an Upgrade After Modifying Values

Next, perform a basic upgrade exercise.

### Modify `values-nginx.yaml`

Change the replica count from 2 to 3, for example:

    replicaCount: 3

Keep other configurations unchanged.

### Execute Upgrade

    helm upgrade my-nginx bitnami/nginx -n helm-demo -f values-nginx.yaml

### Explanation

This step indicates:

- Still use `my-nginx` as the Release
- Still use `bitnami/nginx` as the Chart
- Re-render with new values and update cluster objects

---

## XIV. How to Verify After Upgrade

### 1. Check Release Status Again

    helm status my-nginx -n helm-demo

### 2. Check Current Values Again

    helm get values my-nginx -n helm-demo

### 3. Check Deployment and Pod Again

    kubectl get deploy,pod -n helm-demo

### Expected Phenomenon

You should see:
- Replica count updated to 3
- Increased number of Pods
- Release remains `deployed`

### Operations Understanding Focus

The result of Helm upgrade ultimately affects Kubernetes objects,  
Therefore, the verification logic always includes two layers:

- Helm perspective
- kubectl perspective

---

## XV. What to Check First If Upgrade Fails

Although this article does not expand on rollback practices, at least know the basic troubleshooting entry points.

### 1. Check Helm Status

    helm status my-nginx -n helm-demo

### 2. Check Helm History

    helm history my-nginx -n helm-demo

### 3. Check Kubernetes Object Status

    kubectl get pod -n helm-demo
    kubectl describe pod -n helm-demo <pod-name>
    kubectl logs -n helm-demo <pod-name>

### Explanation

Helm is responsible for:
- Managing the Release lifecycle

But the specific reason for upgrade failure ultimately requires checking:
- Pod
- Deployment
- Service
- Events
- Logs

---

## XVI. Uninstall the Release

After practice, you can uninstall the installed Release.

### Command

    helm uninstall my-nginx -n helm-demo

### Explanation

This step represents:
- Removing the Release `my-nginx` from the namespace `helm-demo`

### Post-Uninstallation Verification

    helm list -n helm-demo
    kubectl get all -n helm-demo

### Expected Observations

Typically, you should see:
- `helm list` no longer contains `my-nginx`
- Corresponding Deployment / Pod / Service resources are cleaned up

### Notes

After actual uninstallation, it's still recommended to check:
- Whether residual objects exist
- Whether PVCs and other resources are deleted

This article uses nginx, which generally doesn't involve PVCs, but this step is crucial in middleware scenarios.

---

## Seventeen. Connecting This Practice into a Complete Closed Loop

The following is the most basic Helm usage closed loop.

### 1. Add Repository

    helm repo add bitnami https://charts.bitnami.com/bitnami

### 2. Update Repository Index

    helm repo update

### 3. Search Chart

    helm search repo nginx

### 4. View Default Values

    helm show values bitnami/nginx > nginx-default-values.yaml

### 5. Prepare Custom Values

    vim values-nginx.yaml

### 6. Install Release

    helm install my-nginx bitnami/nginx -n helm-demo --create-namespace -f values-nginx.yaml

### 7. View Release

    helm list -n helm-demo
    helm status my-nginx -n helm-demo

### 8. View Actual Parameters

    helm get values my-nginx -n helm-demo

### 9. Verify Objects with kubectl

    kubectl get deploy,svc,pod -n helm-demo

### 10. Upgrade After Modifying Values

    helm upgrade my-nginx bitnami/nginx -n helm-demo -f values-nginx.yaml

### 11. Verify Again

    helm status my-nginx -n helm-demo
    kubectl get deploy,pod -n helm-demo

### 12. Uninstall After Practice

    helm uninstall my-nginx -n helm-demo

This workflow can basically serve as a Helm beginner practice template.

---

## Eighteen. Common Misconceptions in This Article

### 1. Only Check Helm After Installation
Correct approach should also check:
- `helm list`
- `helm status`
- `kubectl get ...`

### 2. Assume Cluster Automatically Applies Changes After Modifying values File
The values file is just parameter input; actual effect requires execution of:
- `helm install`
- `helm upgrade`

### 3. Blindly Modify Fields Without First Checking Default values
This easily leads to errors in field paths or parameter names.

### 4. Assume Helm Upgrade Success Equals Business Availability
Upgrade results still need verification:
- Pod
- Service
- Application access
- Logs and events

### 5. Don't Check for Resource Residuals After Uninstallation
Especially in middleware scenarios, residual objects like PVCs, Secrets, CRDs are common.

---

## Nineteen. Key Understandings in This Article

### 1. Helm Practice Focuses on Complete Workflow, Not Individual Commands
This is the first understanding.

### 2. Helm and kubectl Need to Be Used Together
This is the second understanding.

### 3. values File is the Core Entry for Custom Deployment
This is the third understanding.

### 4. Verification Must Be Done After Upgrade
This is the fourth understanding.

### 5. Resource Cleanup Should Be Confirmed After Uninstallation
This is the fifth understanding.

---

## Twenty. Stage Summary

This article connects Helm's basic workflow into a complete closed loop:

- Starting from repository
- To searching Charts
- To viewing default values
- To preparing custom values
- To installing Release
- To verifying objects
- To upgrading changes
- To final uninstallation

Through this practice, Helm's role becomes clearer:

- It's not a replacement for Kubernetes objects
- It organizes and manages a group of objects by application dimension
- It's more suitable for middleware, platform components, and grouped object delivery scenarios

At this point, the **09-Helm and Application Package Management** section has the foundation to proceed to the next stage.

---

## Twenty-one. Keyword Quick Reference

- `helm repo add`: Add repository
- `helm repo update`: Update repository index
- `helm search repo`: Search Chart
- `helm show values`: View default values
- `helm install`: Install Release
- `helm status`: View Release status
- `helm get values`: View actual parameters
- `helm upgrade`: Upgrade Release
- `helm uninstall`: Uninstall Release

---

## Twenty-two. Operational Extended Understanding

The focus of learning Helm in this section is not memorizing several commands, but completing a shift in delivery perspective.

In previous content, the focus was more on:

- How to write objects
- How to combine objects
- How to judge different workload models

At the Helm practice level, the focus shifts to:

- How to install an application
- How to customize an application
- How to upgrade an application
- How to remove an application entirely

This means the perspective has shifted from:

> **Object Definition**

To:

> **Application Delivery and Application Management**

This naturally lays the foundation for the next stage of "Application Release, Update, and Rollback."

---

## References
- Helm Official Documentation
- Kubernetes Official Documentation

---

## Next Day Suggestions
Next stage suggestion: enter

[[01-Application Deployment Basics - Why Release Update and Rollback Are Needed Post-Deployment]]