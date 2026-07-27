# 02-CI-CD Overall Process: Breaking Down a Manual Release into Multiple Phases Including Code, Building, Image Creation, Repository Management, Configuration, and Cluster Deployment

## Document Description

This article is the second notes in the 08-CI-CD learning pathway.

In the previous article, you completed a minimal manual release experiment:

Page file → Build image → Push to Harbor → Deploy to K8s → Re-release after modifications → Observe rolling updates

This article does not introduce new tools. Instead, it focuses on breaking down the steps performed in the previous article into a clear delivery process and re-examining each phase through small experiments.

The goal is not to memorize concepts but to achieve the following two things:

1. Understand the fixed phases involved in a release process.
2. Be able to accurately correlate the actions you performed in the experiment with these phases.

When you later learn about GitLab CI, Jenkins, Helm, and Argo CD, this process will be repeatedly applied.

The default environment for this article is as follows:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Deployment #Image Building #Application Delivery #Practical Notes #Release Process

## Learning Objectives for This Article

After completing this article, you should be able to:

1. Break down a release into 6 fixed phases.
2. Clearly explain the input and output of each phase.
3. Verify each phase using real files and commands in your environment.
4. Identify which phases are automated by GitLab CI/Jenkins.
5. Develop the awareness that "when a release fails, check the corresponding phase of the process first."

## Establishing the Fixed Process for This Article

At this learning stage, let's first identify the 6 fixed phases of the delivery process as follows:

1. Application Content Phase
2. Image Building Phase
3. Image Storage Phase
4. Cluster Configuration Phase
5. Cluster Deployment Phase
6. Release Verification Phase

You can correlate the actions performed in the previous article with these 6 phases:

- `index.html` belongs to the Application Content Phase.
- `docker build` belongs to the Image Building Phase.
- `docker push` belongs to the Image Storage Phase.
- `manual-web.yaml` belongs to the Cluster Configuration Phase.
- `kubectl apply` / `kubectl set image` belongs to the Cluster Deployment Phase.
- `wget` / `rollout status` belongs to the Release Verification Phase.

The goal of this article is to verify each of these 6 phases in detail.

## Experimental Preparation

Continue using the experimental directory from the previous article:

    cd ~/08-ci-cd/01-manual-release

If the directory does not exist, it means the previous experiment was not completed. It is recommended to finish that experiment before proceeding with this one.

## Part 1: Application Content Phase

The core question in this phase is:

**What exactly are you releasing?**

In the previous experiment, the minimal application content actually released was:

    index.html

View the file:

    cat index.html

If you have already updated it to version v2, you should see:

    version: v2

### Input and Output of This Phase

#### Input

- Business content
- Page files
- Source code
- Configuration settings

In this experiment, these correspond to `index.html` and the associated configuration.

#### Output

- A clear indication of "content to be released"

In this experiment, this would be the version tag of the page content, such as:

- `version: v1`
- `version: v2`

### Key Understanding for This Phase

This phase is not related to Kubernetes or Harbor yet. It simply helps determine what exactly will be deployed.

If you switch to a Java project later on, the source code and code changes would correspond to this phase. But for now, use static pages as an example.

## Small Exercise for This Phase

Change the page content to a new version that is easily recognizable. For example:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v2.1</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v2.1</p>
      </body>
    </html>
    EOF

View the result after execution:

    cat index.html

### Observation Points for This Phase

You will notice that:

- Only the local file has been changed.
- The image in Harbor remains unchanged.
- The Pod in K8s also stays the same.
- The cluster access result does not change either.

This shows that:

**Just changing the local file does not- Restart the Pod
- Replace the old Pod with a new one in a rolling manner

## Mini-exercise for this section

It is recommended to directly use `set image` to update it once, as the command is more intuitive:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain-name/test/manual-web:v2.1

Then open three windows to observe the following commands:

    kubectl -n test get deploy -w
    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Observations for this section

You should focus on observing the following:

1. The Deployment does not directly replace the old Pod.
2. A new ReplicaSet will be created.
3. The old and new Pods will coexist for a certain period of time.
4. The old Pod will exit only after the new Pod becomes Ready.

This indicates that:

**The essence of the deployment phase in Kubernetes is to drive rolling updates.**

## Section 6: Release Verification Phase

The core question in this section is:

**How to confirm that this release has actually been successful.**

Don't just rely on the success of `kubectl apply` or the status of Pods being Running.

For the current experiment, perform at least the following two types of verifications.

### Verification 1: Check the rollout status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Verification 2: Verify the actual business content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering the command, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Input and Output for this section

#### Input

- The current status of the Deployment
- The current status of the Pods
- The result of accessing the Service

#### Output

- A judgment on whether a release has truly been completed

### Understanding to be established in this section

Here, two levels need to be distinguished:

#### Level 1: From the perspective of the Kubernetes controller

If `rollout status` shows success, it means that:

- The update process has converged.
- The new Pod has started as expected.

#### Level 2: From the business perspective

Only when you actually see `version: v2.1` on the accessed page can you confirm that:

- The new version of the content has truly taken effect externally.

Therefore:

**The release verification phase is not just about checking whether a single command succeeds, but also about examining the cluster status and business results.**

## Mini-exercise for this section

After completing these steps, record the following two outputs:

1. The output of `kubectl -n test rollout status deployment/manual-web`
2. The output of `wget -qO- http://manual-web.test.svc.cluster.local`

### Observations for this section

Only when you see both:

- Successful rollout
- The page content being v2.1

can you consider that the release has truly been completed.

## Reconnecting all steps together

Now, reassemble the actions you just performed into a fixed sequence:

### 1) Application Content Phase

You modified:

- `index.html`

### 2) Image Building Phase

You executed:

- `docker build -t manual-web:v2.1 .`

### 3) Image Storage Phase

You executed:

- `docker tag ...`
- `docker push ...`

### 4) Cluster Configuration Phase

You checked or modified:

- `manual-web.yaml`
- The `image` field within it

### 5) Cluster Release Phase

You executed:

- `kubectl apply -f ...`
- Or `kubectl set image ...`

### 6) Release Verification Phase

You executed:

- `kubectl rollout status ...`
- `wget -qO- http://manual-web.test.svc.cluster.local`

By doing this, you have clearly identified the entire release process.

## Now, reflect on what GitLab CI / Jenkins will automate in the future

For now, let's not discuss the configuration of GitLab CI / Jenkins; instead, focus on the actions you have already performed.

In the future, pipeline automation will mainly involve these fixed steps:

### Parts that GitLab CI / Jenkins will automate

1. Pulling code
2. Building images
3. Adding tags
4. Pushing to Harbor
5. Updating configurations or executing release commands
6. Performing basic verifications

You don't need to rush into learning their syntax right now.  
First, thoroughly master this manual process before exploring pipelines, and you will find it less abstract.

## Content that should be able to be explained after completing this chapter

After going through this chapter, you should be able to clearly explain the following:

A release can be broken down into 6 fixed phases: Application Content, Image Building, Image Storage, Cluster Configuration, Cluster Release