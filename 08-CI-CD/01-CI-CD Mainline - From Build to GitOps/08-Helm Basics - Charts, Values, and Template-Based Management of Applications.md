# 08-Helm Basics: Charts, Values, and Template-Based Management of Applications

## Document Description

This article is the eighth note in the 08-CI-CD learning pathway.

In the previous articles, we have covered the following key topics:

1. Manually implementing the minimum release pipeline
2. Breaking down the release process into fixed stages
3. Understanding image building and image tag design
4. Comprehending projects, repositories, tags, and Robot accounts in Harbor
5. Mapping manual tasks to GitLab CI
6. Mapping manual tasks to Jenkins Pipeline
7. Grasping Deployment, ReplicaSet, Pod, and rolling update mechanisms

In this article, we will focus on solving an issue that has become increasingly prominent:

**How to manage, reuse, and release these YAML files more efficiently when an application has multiple Deployments and Services?**

This article continues the "learn by doing" approach used in previous sections. Instead of introducing many concepts first, we will directly transform your existing native YAML files into a minimal Helm Chart and demonstrate how to perform the following operations:

- `helm install`
- `helm upgrade`
- `helm rollback`

This article remains consistent with the current learning environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Helm #Chart #Values #Template-Based #K8s Release #Rollout #rollback #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what Helm is primarily designed to solve
2. Define what Charts, templates, and values are
3. Recognize why Helm is suitable for multiple environments and repeated releases
4. Transform existing Deployment + Service YAML files into a minimal Helm Chart
5. Perform a `helm install`
6. Modify values and perform a `helm upgrade`
7. View the release history and execute a `helm rollback`

## First, Reconnect Helm with the Pipeline You Have Learned

You have already gone through this pipeline in previous manual experiments:

Application content  
→ Image building  
→ Image storage  
→ Cluster configuration  
→ Cluster deployment  
→ Release verification

The methods you used for "cluster configuration" mainly involved:

- Manually writing `manual-web.yaml`
- Manually modifying the image
- Running `kubectl apply`
- Or using `kubectl set image`

This approach works well when there are few resources, but it becomes increasingly difficult to maintain as the number of resources and environments increases.

Therefore, Helm can be understood in this learning context as:

**It represents a templated and engineered enhancement for the "cluster configuration phase" and the "cluster deployment phase."**

---

## Part 1: Identifying Issues with Native YAML

Let's return to the files you have used before:

    cd ~/08-ci-cd/01-manual-release
    ls
    cat manual-web.yaml

The `manual-web.yaml` file you see usually contains fixed values such as:

- Application name
- Namespace
- Image address
- Number of replicas
- Service name
- Label / selector

### Understanding Required for This Step

When there is only one environment and two resources, this approach works fine.

However, if you have additional environments such as `test` or `prod`, different image tags, varying numbers of replicas, different domain names, or more resource objects, you will encounter the following problems:

1. You need to create multiple copies of the same YAML file.
2. It's easy to miss changes when modifying the image.
3. Modifications to the name, label, or selector can lead to inconsistencies.
4. Rollback and version management become less straightforward.

Helm is designed to address these issues.

---

## Part 2: Setting Up the Helm Environment

### Step 1: Confirming Helm Installation

Run:

    helm version

### Expected Outcome

You should see the Helm version information, for example:

    version.BuildInfo{Version:"v3.x.x", ...}

If Helm is not installed, pause this article and complete the installation before continuing.

### Understanding Required for This Step

All subsequent `helm install / upgrade / rollback` commands rely on the Helm command-line tool, so Helm must be installed on your system first.

---

## Part 3: Creating a Minimal Helm Chart

### Step 1: Entering the Helm Experiment Directory

It is recommended to create a new directory for this purpose:

    mkdir -p ~/08-ci-cd/08-helm-lab
    cd ~/08-ci-cd/08-helm-lab

### Step 2: Using Helm to Create the Chart Skeleton

Run:

    helm create manual-web

Enter the directory:

    cd manual-web

View the directory structure:

    tree .

If you don't see the `tree` command, you can    cat templates/service.yaml

### Step 2: Pay attention to the following two fields

- service type
- service port

### Understanding required for this step

Similar to Deployment:

- The Service template determines the resource structure.
- The values determine parameters such as the port and type.

By now, you should have a basic understanding of Helm:

**Helm = Templates + Parameters**

---

## Section 9: Don't install yet, check the rendered output first

This is a very important step.

Before actually executing `helm install`, first look at the YAML that Helm renders.

### Step 1: Execute template rendering

In the root directory of the `manual-web` Chart, execute:

    helm template manual-web .

### Expected outcome

A large amount of YAML will be output.

### What to focus on in this step

You need to confirm:

1. Whether there is a Deployment in the output.
2. Whether there is a Service in the output.
3. Whether the `image` has been changed to the Harbor address you specified in values.
4. Whether the `replicas` have been set to 2.
5. Whether the Service port is 80.

### Understanding required for this step

This step is crucial because it shows that:

**Helm ultimately generates regular K8s YAML.**

In other words, Helm does not work independently of YAML; instead, it adds an extra layer before generating YAML:

- Templates
- Parameter rendering
- Release management

---

## Section 10: Officially install the Helm Release

### Step 1: Execute installation

Still in the root directory of the Chart, execute:

    helm install manual-web . -n test --create-namespace

### Expected outcome

You will see output similar to this:

    NAME: manual-web
    NAMESPACE: test
    STATUS: deployed

### Understanding required for this step

Here are two key concepts:

#### Chart

The set of template files in the current directory.

#### Release

The `manual-web` instance that is installed.

So, what happened here is not just applying some YAML files; instead,

**a Chart was installed as a Release named `manual-web`.**

---

## Section 11: Confirm that the Release has taken effect

### Step 1: View the release list

Execute:

    helm list -n test

### Step 2: View Deployment and Pod

Execute:

    kubectl -n test get deploy
    kubectl -n test get pods
    kubectl -n test get svc

### Step 3: View the page content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside that command, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected outcome

You should see the version number corresponding to the values you specified, for example:

    version: v3

### Understanding required for this step

By now, you should be clear about the following:

- `helm install` ultimately creates Deployments and Services in the cluster.
- It doesn't directly use `kubectl apply manual-web.yaml`.
- Instead, it first renders the templates and then uses Helm for release management.

---

## Section 12: Perform an upgrade by modifying values

This section is one of the most critical practices in this guide.

### Step 1: Prepare a new image tag

Assume you have already pushed a new tag earlier, such as:

- `dev-c1d2e3f-301`
- Or any other tag you used during your practice.

### Step 2: Modify the `image.tag` in values.yaml

Execute:

    grep -n "tag:" values.yaml

Then change:

    tag: "v3"

to the new tag that actually exists in your Harbor repository, for example:

    tag: "dev-c1d2e3f-301"

Save the file and confirm again:

    cat values.yaml

### Step 3: Execute the upgrade

Execute:

    helm upgrade manual-web . -n test

### Step 4: Observe the update process

It is recommended to open two separate windows to execute the following commands:

    kubectl -n test get rs -w
    kubectl -n test get pods -w

Then execute:

    kubectl -n test rollout status deployment/manual-web -n test

### Step 5: Verify the page content again

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside that command, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding required for this step

The key point here is:

**You didn't directly modify the Deployment YAML this time; instead, you only changed the values and then executed `helm upgrade`.**

Helm then performed the following actions:

-### 1. What problems does Helm mainly address?

Helm primarily addresses the need for templating and parameterized management of Kubernetes resources. As the number of application resources increases, maintaining numerous native YAML files becomes increasingly difficult. Helm helps separate the resource structure from variable parameters through Chart, templates, and values. By using these tools, Helm simplifies the process of managing and deploying resources.

### 2. What are Chart, templates, and values?

- **Chart**: It represents the entire application package, containing the definition of the resources that need to be deployed.
- **Templates**: These are files that define the structure of the resources, including their specifications and configurations.
- **Values**: They provide the dynamic data that can be used to customize these templates. These values can come from various sources, such as environment variables or external files.

### 3. What is the role of `helm template`?

The `helm template` serves as a blueprint for creating Kubernetes resources. It defines the structure and configuration of the resources, while the `values` file provides the specific values that will be used to populate these templates during deployment.

### 4. What is the difference between `helm upgrade` and manual `kubectl apply`?

- **`helm upgrade`**: It updates a Kubernetes resource package based on new template definitions or changed values. This process includes creating a new Release, applying the updated resources, and then rolling back any changes if necessary.
- **Manual `kubectl apply`**: This command applies a specific YAML file directly to the Kubernetes cluster, without going through the Helm pipeline. It provides more control over the deployment process but may require more manual effort.

### 5. What exactly is the object that `helm rollback` rolls back to?

`helm rollback` restores a Kubernetes resource package to a previous version based on the stored Release history. The specific objects that are rolled back depend on the type of resource being managed. For example, if you roll back a Deployment, it will revert all the related pods and services to their previous configurations.

If you can clearly explain these 5 questions, you have a good understanding of Helm and its fundamental concepts.