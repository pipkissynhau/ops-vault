# 08-Helm Basics: Chart, Values, and Application Templating Management

## Document Notes

This article is the 8th note in the 08-CI-CD learning path.

The previous articles have covered these main topics:

1. Manually walk through the minimal release pipeline
2. Break down the release into fixed stages
3. Understand image building and image Tag design
4. Understand projects, repositories, tags, and Robot accounts in Harbor
5. Map manual actions to GitLab CI
6. Map manual actions to Jenkins Pipeline
7. Understand Deployment, ReplicaSet, Pod, and rolling update mechanisms

This article focuses on solving an increasingly apparent problem:

**When application resources are no longer just a single Deployment and Service, how to manage, reuse, and release these YAMLs more efficiently.**

This article continues using the "do-while-understanding" approach from before, not piling up many concepts first, but directly transforming your existing native YAML into a minimal Helm Chart and completing:

- helm install
- helm upgrade
- helm rollback

This article continues to align with the current learning environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Helm #Chart #Values #Templateization #K8sRelease #rollout #rollback #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understand what Helm primarily solves
2. Understand what Chart, templates, and values are respectively
3. Understand why Helm is suitable for multi-environment and repeated releases
4. Be able to transform existing Deployment + Service YAML into a minimal Helm Chart
5. Be able to complete a `helm install`
6. Be able to modify values and complete a `helm upgrade`
7. Be able to view release history and execute a `helm rollback`

## First, Put Helm Back Into Your Learned Pipeline

Previously, you've walked through this chain manually:

Application Content  
→ Image Building  
→ Image Repository  
→ Cluster Configuration  
→ Cluster Deployment  
→ Deployment Verification

Previously, your "cluster configuration" approach was mostly:

- Manually writing `manual-web.yaml`
- Manually changing image
- `kubectl apply`
- Or `kubectl set image`

This approach is fine when resources are few, but becomes increasingly difficult to maintain when resources and environments grow.

So, Helm's position in the current learning pipeline can be initially understood as:

**It is a templating and engineering enhancement for the "cluster configuration" and "cluster deployment" stages.**

---

## Part One: First, Identify Issues With Native YAML

Return to the file you've used previously:

    cd ~/08-ci-cd/01-manual-release
    ls
    cat manual-web.yaml

You now see `manual-web.yaml`, which typically contains these hard-coded elements:

- Application name
- namespace
- image address
- replica count
- Service name
- label / selector

### Understanding to Establish at This Stage

If there's only one environment and two resources, writing like this is acceptable.

But if you later have:

- `test`
- `prod`
- Different image tags
- Different replica counts
- Different domain names
- More resource objects

You'll easily encounter these issues:

1. A single YAML needs to be copied many times
2. Modifying image addresses may easily be missed
3. Name / label / selector changes may become inconsistent
4. Rollback and version management are unclear

Helm solves exactly these types of problems.

---

## Part Two: Prepare Helm Environment

### Step 1: Confirm Helm is Installed

Execute:

    helm version

### Expected Phenomenon

Normally, you'll see Helm version information, for example:

    version.BuildInfo{Version:"v3.x.x", ...}

If Helm is not installed, this article will pause until you complete Helm installation before continuing.

### Understanding to Establish at This Stage

All subsequent `helm install / upgrade / rollback` are based on Helm CLI tools, so Helm must be installed on this machine.

---

## Part Three: Create a Minimal Helm Chart

### Step 1: Enter Helm Experiment Directory

Recommended to create a new directory:

    mkdir -p ~/08-ci-cd/08-helm-lab
    cd ~/08-ci-cd/08-helm-lab

### Step 2: Use Helm to Create Chart Skeleton

Execute:

    helm create manual-web

Enter the directory:

    cd manual-web

Check the directory structure:

    tree .

If there's no `tree`, you can use:

    find .

### Expected Phenomenon

You'll see a basic Helm Chart skeleton, typically containing:

- `Chart.yaml`
- `values.yaml`
- `templates/`

### Understanding to Establish at This Stage

You don't need to memorize all directories at first. Just remember three things:

- `Chart.yaml`: Chart's own metadata
- `values.yaml`: Default parameters
- `templates/`: Resource templates

These three are the core components of Helm at this stage.

---

## Part Four: First, See What the Default Chart Looks Like

### Step 1: View Chart.yaml

Execute:

    cat Chart.yaml

### Step 2: View values.yaml

Execute:

    cat values.yaml

### Step 3: View templates Directory

Execute:

    ls templates

### Understanding to Establish at This Stage

Helm's default generated Chart will contain much more than what your current experiment needs.  
Currently, you don't need to absorb everything at once. Instead, first do this:

**Shrink it into the minimal version suitable for your current experiment.**

---

## Part Five: Minimize the Default Chart to What Your Experiment Needs

Your current experiment only needs:

- Deployment
- Service

So first delete the unnecessary template files.

### Step 1: Enter the templates Directory and View Files

Execute:

    ls templates

### Step 2: Delete Templates Not Needed for Your Experiment

Typically, you can keep:

- `deployment.yaml`
- `service.yaml`
- `_helpers.tpl`

Delete others first, such as: /think

```markdown
rm -f templates/ingress.yaml
rm -f templates/hpa.yaml
rm -f templates/serviceaccount.yaml
rm -f templates/NOTES.txt
rm -f templates/tests/test-connection.yaml

If a file is not present in the directory, skip it.

### Step 3: Verify the templates directory again

Execute:

    ls templates

### Expected outcome

Try to keep only:

- `deployment.yaml`
- `service.yaml`
- `_helpers.tpl`

### Understanding this step

Helm does not require you to use all templates from the start.  
Currently, the most important thing is:

**Shrink the Chart to the minimal set you can actually understand and use.**

---

## Part 6: Modify values.yaml to retain only critical parameters needed for the current experiment

### Step 1: Open values.yaml

Execute:

    cat values.yaml

The default content will be extensive. At this stage, you don't need such complexity, so it's recommended to directly change it to the minimal version below.

Execute:

    cat > values.yaml <<'EOF'
    replicaCount: 2

    image:
      repository: Yours.HarborDomain Name/test/manual-web
      tag: "v3"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    containerPort: 80

    namespace: test
    EOF

Replace `Yours.HarborDomain Name` with your actual Harbor address.

### Understanding this step

Now the role of `values.yaml` is very clear:

- Place replica count here
- Place image address here
- Place image tag here
- Place Service port here
- Place namespace here

When doing upgrades later, the main changes will be to these values.

This is a key difference between Helm and directly modifying native YAML:

**Many changes no longer require modifying the templates themselves, but instead modify values.**

---

## Part 7: Check how Deployment templates reference values

### Step 1: View the deployment template

Execute:

    cat templates/deployment.yaml

The default template will be slightly complex, but at this stage, you mainly focus on these points:

- name
- replicas
- image
- containerPort

### Step 2: Search for image-related content in the template

Execute:

    grep -n "image" -n templates/deployment.yaml

### Step 3: Search for replica count-related content

Execute:

    grep -n "replica" -n templates/deployment.yaml

### Understanding this step

You don't need to fully master Helm template syntax right now. Just confirm one thing:

**deployment.yaml is not a static YAML file, but references values in `values.yaml`.**

In other words:

- Templates define the structure
- Values define specific parameters

---

## Part 8: Check how Service templates reference values

### Step 1: View the service template

Execute:

    cat templates/service.yaml

### Step 2: Focus on these two fields

- service type
- service port

### Understanding this step

This is the same as with Deployment:

- Service templates define resource structure
- values define parameters like port and type

At this point, you should be able to form the most basic understanding of Helm:

**Helm = Templates + Parameters**

---

## Part 9: Don't install yet, first check the rendered output

This is a very important step.

Before actually `helm install`, first check the YAML output rendered by Helm.

### Step 1: Execute template rendering

In the root directory of the `manual-web` Chart, execute:

    helm template manual-web .

### Expected outcome

A long YAML output will be generated.

### What to focus on in this step

You need to confirm:

1. Does the output contain Deployment?
2. Does the output contain Service?
3. Has the image been changed to your Harbor address in values?
4. Has replicaCount been changed to 2?
5. Is the Service port 80?

### Understanding this step

This step is crucial because it shows:

**Helm ultimately still generates regular K8s YAML.**

In other words, Helm isn'tGraduation YAML, but adds an extra layer before generating YAML:

- Templates
- Parameter rendering
- Release management

---

## Part 10: Officially install the Helm Release

### Step 1: Execute install

Still in the Chart root directory, execute:

    helm install manual-web . -n test --create-namespace

### Expected outcome

You'll see output like:

    NAME: manual-web
    NAMESPACE: test
    STATUS: deployed

### Understanding this step

There are two key concepts here:

#### Chart

The template package in the current directory.

#### Release

`manual-web` This is the instance created by the installation.

So now it's not simply "applying several YAML files," but:

**Installing a Chart as a Release called `manual-web`.**

---

## Part 11: Confirm the Release is active

### Step 1: View the release list

Execute:

    helm list -n test

### Step 2: View Deployment and Pod

Execute:

    kubectl -n test get deploy
    kubectl -n test get pods
    kubectl -n test get svc

### Step 3: View page content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected outcome

You should see the content corresponding to the version in values, for example:

    version: v3
```

### Understanding to be Established at This Step

By now you should clearly understand:

- Helm install ultimately still creates Deployment / Service in the cluster
- It's not directly `kubectl apply manual-web.yaml`
- Instead it first renders templates through Helm, then manages the release

---

## Part 12: Completing an Upgrade by Modifying values

This is one of the most critical practical exercises in this article.

### Step 1: Prepare a New Image Tag

Assume you've already pushed a new tag, for example:

- `dev-c1d2e3f-301`
- Or another tag you pushed during practice

### Step 2: Modify image.tag in values.yaml

Execute:

    grep -n "tag:" values.yaml

Then change:

    tag: "v3"

to your actual new tag in Harbor, for example:

    tag: "dev-c1d2e3f-301"

Save and confirm:

    cat values.yaml

### Step 3: Execute Upgrade

Execute:

    helm upgrade manual-web . -n test

### Step 4: Observe the Update Process

Recommend running two windows separately:

    kubectl -n test get rs -w
    kubectl -n test get pods -w

Then execute:

    kubectl -n test rollout status deployment/manual-web -n test

### Step 5: Verify Page Content Again

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding to be Established at This Step

Here you should focus on seeing a key fact:

**You didn't directly modify the Deployment YAML this time, but only changed values and executed helm upgrade.**

Then Helm completed:

- Re-rendering templates
- Updating cluster resources
- Triggering Deployment rolling update

This is one of Helm's core values:

**Configuration changes no longer scatter across many YAML files, but are managed centrally in values.**

---

## Part 13: Viewing Helm Release History

### Step 1: View Release History

Execute:

    helm history manual-web -n test

### Expected Phenomenon

You'll see records like:

- revision 1: install
- revision 2: upgrade

### Understanding to be Established at This Step

This is somewhat similar to Deployment's own rollout history, but with a higher perspective.

Deployment's history focuses more on individual Deployment template history.  
Helm's history represents:

**The complete release history of the entire application.**

This means Helm is particularly suitable for "application-level" upgrades and rollbacks.

---

## Part 14: Executing a Rollback

To truly understand Helm's value, this step is strongly recommended.

### Step 1: Rollback to revision 1

Execute:

    helm rollback manual-web 1 -n test

### Step 2: View Release History

Execute:

    helm history manual-web -n test

### Step 3: Observe Pod Update Process

Execute:

    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Step 4: Verify Page Content Again

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding to be Established at This Step

Here you should see:

- Helm rollback is not an abstract concept
- It ultimately drives K8s to update Deployment
- The entry point isn't manually editing YAML, but "rolling back release history versions"

So Helm's rollback capability essentially means:

**Rolling back the entire template and parameter state history at the release level.**

---

## Part 15: Now Re-understand Chart, templates, values

By now, understanding these three concepts becomes much easier.

### 1) Chart

The current `manual-web` directory as a whole is a Chart.

It is:

- An application template package
- Can be installed
- Can be upgraded
- Can be rolled back

### 2) templates

The files in the `templates/` directory are resource templates.

They define:

- How a Deployment looks
- How a Service looks

### 3) values

The values in `values.yaml` are the source of template parameters.

It controls:

- Image repository
- Image tag
- Replicas
- Service port

### Understanding to be Established at This Step

You should now naturally be able to state:

**Helm = Using values to render templates into final YAML, then publishing as a Release to K8s.**

---

## Part 16: Why Helm is More Suitable for Subsequent Learning Mainline Than Manual YAML Maintenance

Combined with your current experiments, this point becomes easy to understand.

### Without Helm

In the future, each release might require:

- Modifying many YAML files
- Changing image
- Changing replicas
- Changing Service parameters
- Manual apply
- Manual maintenance of environment copies

### With Helm

Often just need:

- Modifying values
- helm upgrade

This makes subsequent capabilities smoother:

- GitLab CI calling Helm
- Jenkins Pipeline calling Helm
- Multi-environment values management
- Release history viewing
- Quick rollback

So Helm isn't replacing K8s, but making "configuration management and release execution" more engineering-oriented.

---

## Part 17: This Article's Small Exercise

### Exercise 1: Do an Upgrade Again

Requirements:

- Prepare another real new tag in Harbor
- Only modify `values.yaml`'s `image.tag`
- Execute `helm upgrade`
- Verify changes in page content

### Exercise 2: Do a Rollback Again

Requirements:

- View `helm history`
- Rollback to the previous revision
- Verify if the page has reverted to the old version

### Exercise 3: Answer the following 5 questions yourself

1. What problem does Helm mainly solve?
2. What are Chart, templates, and values respectively?
3. What is the purpose of `helm template`?
4. What is the difference between `helm upgrade` and manual `kubectl apply`?
5. What exactly is the rollback target of `helm rollback`?

If you can explain these 5 questions clearly by yourself, you've mastered this chapter.

---

## Content to be able to explain after completing this chapter

After finishing this chapter, it's recommended to be able to explain the following passage yourself:

Helm mainly solves the problem of templating and parameterized management of Kubernetes resources.  
When application resources grow increasingly complex, directly maintaining many native YAML files becomes harder. Helm separates resource structure from variable parameters through Chart, templates, and values.  
Chart is the entire application package, templates are resource templates, and values are the source of template parameters.  
`helm install` is installing a Chart as a Release, `helm upgrade` is upgrading this Release based on new values or template content, and `helm rollback` is rolling back the Release to a historical version.  
Therefore, Helm is suitable forTake over. previous manual configuration and deployment actions, and lays the foundation for subsequent GitLab CI, Jenkins, and more complex multi-environment deployments.

## Common Issues and Troubleshooting Directions

### Problem 1: `helm template` output is hard to understand

At this stage, don't try to understand all details at once.  
Focus first on:

- image
- replicas
- Service port
- name

Confirm whether values have been rendered properly.

### Problem 2: `helm install` succeeded but Pod hasn't started

Essentially the same as troubleshooting manual deployment failures:

- Is the image address correct?
- Does the tag exist in Harbor?
- Does containerd trust Harbor?
- Is private repository authentication available?

Check the commands:

    kubectl -n test get pods
    kubectl -n test describe pod PodName

### Problem 3: `helm upgrade` didn't change the page

Prioritize checking:

- Has image.tag in values.yaml really changed?
- Does the tag exist in Harbor?
- Has Deployment really updated the image?
- Does Service access the new Pod?

### Problem 4: Why do you still see Pod updates after `helm rollback`?

Because Helm rollback ultimately still updates K8s resources.  
After receiving old version configuration, Deployment will trigger rolling update again.

---

## Key Takeaways of This Chapter

After completing this chapter, you should master:

1. Helm's position in the current learningMain
2. Why templating management is needed when resources grow
3. The relationship between Chart, templates, and values
4. How to convert existing native YAML into a minimal Helm Chart
5. How to execute `helm install`
6. How to perform `helm upgrade` by modifying values
7. How to view release history and execute `helm rollback`

## One-Sentence Summary

Helm's essence is to turn a group of native Kubernetes resources into a reusable, parameterizable, and rollable-back application package, making configuration management and deployment execution more engineering-oriented.

## Next Chapter

Next chapter enters:

09-CI-CD DeploymentActual: From Building Images to Deploying to Kubernetes

The next chapter will truly connect the previous 01 to 08 content into a completeMain, with focus on:

- Starting from application content
- Build image
- Push to Harbor
- Choose between kubectl or Helm for deployment
- Rollout verification
- Segment-by-segment troubleshooting along the chain when issues occur