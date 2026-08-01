# 13 - Multi-Environment Deployment Basics: Splitting dev, test, prod Configuration Strategies

## Document Notes

This is the 13th note in the 08-CI-CD learning path.

The previous 01-12 notes have already established a minimal deployment pipeline and completed mirror tag and mirror pull mechanisms.  
This note continues the journey, addressing a very practical issue in a deployment system:

**Why must dev, test, and prod be distinguished when an application enters team collaboration and continuous deployment phases?**

Many beginners in K8s or CI/CD start with a single environment, with everything written together, and it can still run.  
But once the following appear:

- Development verification
- Joint debugging testing
- Stable release
- Formal launch

These stages with different target goals, environment splitting becomes necessary.

This article's focus is not to fully set up three environments at once, but to first establish a **practical, maintainable, and suitable for the current stage** multi-environment strategy.

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- Existing `test` namespace
- Existing minimal application `manual-web`

## Tags

#Kubernetes #CI-CD #Multi-environmentRelease #dev #test #prod #Helm #values #Namespace #ConfigureSplit #I'llTakeYourNotes.

## Learning Objectives

After completing this note, you should understand:

1. Why dev, test, prod need to be split
2. What are the most common differences in multi-environment setups
3. What should be prioritized and what should not be over-designed at this stage
4. How to simulate dev/test environments in the current cluster using namespaces
5. How to use different mirror tags and different replica counts for two environments
6. Why Helm values are particularly suitable for multi-environment management
7. To explain clearly that "multi-environment is not multiple systems, but the same application with different parameter combinations for different goals"

## This Note's Experimental Mainline

This note is divided into 4 sections:

1. First establish the minimal understanding of multi-environment
2. Simulate dev/test environments in the current cluster
3. Complete two deployments with different tags and replica counts
4. Organize multi-environment parameters using Helm values approach

---

## Part 1: Why Split dev, test, prod

Without discussing comprehensive platforms, let's look at the most practical issues first.

Assume you currently have only one `test` environment, and the following needs appear:

- Developers want to quickly verify new changes
- Testers want to use a relatively stable version for verification
- You want to preserve a version that "more closely resembles formal release"

If these needs are all mixed in one environment, several problems will immediately arise:

### Problem 1: Different goals interfere with each other

- Developers want speed
- Testers want stability
- Pre-launch verification wants traceability

But a single environment cannot satisfy all these simultaneously.

### Problem 2: Version meaning becomes confusing

Today's deployment could be "development temporary version", "test version", or "pre-production version", making it easy to get confused.

### Problem 3: Configuration needs differ

Different environments often have different:

- Replica counts
- Tags
- Access entry points
- Resource limits
- Dependency configurations

If everything is combined, it will become increasingly chaotic.

### Understanding to Establish at This Stage

Splitting dev, test, prod is not to make the system more complex,  
but to enable:

- Clearer goals
- More controllable versions
- More manageable configurations

---

## Part 2: Understand the Focus of the Three Types of Environments in One Sentence

At this stage, first establish the most practical understanding without getting too complex.

### dev

Favors development verification:

- Fast updates
- Frequent changes
- High fault tolerance
- Doesn't pursue long-term stability

### test

Favors testing and joint debugging:

- Needs certain stability
- Needs repeatable verification
- Versions should be relatively clear

### prod

Favors formal services:

- Stability first
- Cautious changes
- Versions must be traceable
- Rollbacks must be clear

### Understanding to Establish at This Stage

The three are not "technically completely different three systems",  
but:

**The same application using different versions and parameters for different goals.**

---

## Part 3: What Are the Most Common Differences in Multi-Environment Setup

At this stage, don't need to split all differences, just focus on the most common 5 categories.

### 1) Mirror Tag

For example:

- dev: Can be more frequent, biased toward development line
- test: Biased toward test version
- prod: Biased toward formal version or release version

### 2) Replica Count

For example:

- dev: 1
- test: 1 or 2
- prod: More than 2 is common

### 3) Namespace

For example:

- dev
- test
- prod

This is the easiest and most suitable way to isolate environments at this stage.

### 4) Access Entry Point

For example:

- dev: Internal verification entry point
- test: Test domain name
- prod: Formal domain name

At this stage, if you haven't reached Ingress yet, just establish this understanding.

### 5) Configuration Parameters

For example:

- Database address
- Redis address
- Third-party interface address
- Resource limits
- HPA switch

### Understanding to Establish at This Stage

The essence of multi-environment deployment is not replicating three systems,  
but:

**The same application template + different environment parameters.**

---

## Part 4: Don't Over-Design at This Stage

This is very important.

You are currently in the learning and experimentation phase, and it's not recommended to do the following heavy designs upfront:

- Three completely different YAML directories
- A lot of duplicated configurations for each environment
- Splitting many scripts and templates from the start
- Setting up a complete GitOps multi-repository structure from the start

At this stage, the most suitable first steps are:

1. Use namespaces to distinguish environments
2. Use mirror tags to distinguish version lines
3. Use a few parameters to distinguish replica counts and deployment goals
4. Later use Helm values to consolidate environment differences

### Understanding to Establish at This Stage

Multi-environment needs to "have boundaries", but doesn't need to "over-engineer".

---

## Part 5: Hands-on - Simulate dev and test Environments in the Current Cluster

This section starts the practical work.

### Step 1: Confirm the Existence of the test Environment

Execute:

    kubectl get ns

If you already have `test`, proceed.

### Step 2: Create dev Namespace

Execute:

    kubectl create namespace dev

If it prompts that it already exists, you can ignore it.

### Step 3: Verify the Namespaces Again

Execute:

    kubectl get ns

### Understanding to Establish at This Stage

The simplest and most effective way to isolate environments at this stage is:

- `dev` namespace
- `test` namespace

This way, you can at least place two environment instances in the same cluster later without them overlapping.

cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release dev</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: dev-13</p>
      </body>
    </html>
    EOF

Build and push:

    docker build -t manual-web:dev-13 .
    docker tag manual-web:dev-13 your Harbor domain name/test/manual-web:dev-13
    docker push your Harbor domain name/test/manual-web:dev-13

### Second Step: Prepare test version content

Change to test version:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release test</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: test-13</p>
      </body>
    </html>
    EOF

Build and push:

    docker build -t manual-web:test-13 .
    docker tag manual-web:test-13 your Harbor domain name/test/manual-web:test-13
    docker push your Harbor domain name/test/manual-web:test-13

### Current Understanding to Establish

This is not about "two completely different applications", but:

- Same application
- Two different environment target versions

That is, the dev environment and test environment later are still `manual-web`, but:

- Use different namespace
- Use different image tag

---

## Seventh Section: Hands-on - Prepare a Minimal YAML for dev Environment

At this stage, don't pursue templating, just directly create a dev version YAML.

### First Step: Create dev Environment YAML

Execute:

    cat > manual-web-dev.yaml <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: manual-web
      namespace: dev
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: manual-web
      template:
        metadata:
          labels:
            app: manual-web
        spec:
          containers:
            - name: manual-web
              image: your Harbor domain name/test/manual-web:dev-13
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: manual-web
      namespace: dev
    spec:
      selector:
        app: manual-web
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP
    EOF

### Current Understanding to Establish

Here we simulate the dev environment with minimal differences:

- namespace = dev
- replicas = 1
- image = dev-13

This is a very typical "development verification environment" configuration.

---

## Eighth Section: Hands-on - Prepare a Minimal YAML for test Environment

### First Step: Create test Environment YAML

Execute:

    cat > manual-web-test.yaml <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: manual-web
      namespace: test
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: manual-web
      template:
        metadata:
          labels:
            app: manual-web
        spec:
          containers:
            - name: manual-web
              image: your Harbor domain name/test/manual-web:test-13
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: manual-web
      namespace: test
    spec:
      selector:
        app: manual-web
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP
    EOF

### Current Understanding to Establish

Here we simulate the test environment with minimal differences:

- namespace = test
- replicas = 2
- image = test-13

You now have the feeling of "the same application with different environment target parameters."

---

## Part 9: Hands-on - Deploy dev and test Separately

### Step 1: Deploy dev Environment

Execute:

    kubectl apply -f manual-web-dev.yaml

Check:

    kubectl -n dev get deploy,pods,svc

### Step 2: Deploy test Environment

Execute:

    kubectl apply -f manual-web-test.yaml

Check:

    kubectl -n test get deploy,pods,svc

### Current Step Understanding

At this point, you already have two environments in the same cluster:

- dev / manual-web
- test / manual-web

They:

- Can have the same name
- Can have the same resource logic
- But are isolated through namespace and image tag

This is the most basic practice of multi-environment deployment.

---

## Part 10: Hands-on - Verify Business Results for dev and test

### Verify dev

Execute:

    kubectl -n dev run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.dev.svc.cluster.local

Expected to see:

    version: dev-13

### Verify test

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

Expected to see:

    version: test-13

### Current Step Understanding

This step is very important because it lets you see:

- The same application name
- In different namespaces
- Can really host different environment versions simultaneously

This is the core intuitive experience of multi-environment deployment.

---

## Part 11: Now Summarize the Current Minimal Differences Between dev/test

You have now manually created two environments. You can now summarize the current differences.

### dev Current Differences

- namespace: `dev`
- replicas: 1
- image tag: `dev-13`

### test Current Differences

- namespace: `test`
- replicas: 2
- image tag: `test-13`

### Current Step Understanding

This is the minimal feasible split for multi-environment at this stage:

- You don't need to split all parameters from the beginning
- Just split out the most core differences first is enough

---

## Part 12: Why Helm Values Are Particularly Suitable for Multi-Environment

At this point, looking back at Helm naturally makes sense.

Without Helm, you would now need to maintain two YAML files:

- `manual-web-dev.yaml`
- `manual-web-test.yaml`

As the number of files grows, you'll quickly find:

- A lot of repeated content
- Need to modify both when changing structure
- Prone to errors

This is where Helm's value becomes evident:

### Same Template

Keep a single Deployment/Service template.

### Different Environment Values

For example:

- `values-dev.yaml`
- `values-test.yaml`
- `values-prod.yaml`

This concentrates the differences into parameter files.

### Current Step Understanding

Multi-environment isn't something Helm can do alone,  
but:

**Helm is particularly suitable for multi-environment.**

Because it naturally supports:

- Template reuse
- Parameter overriding
- Multi-environment values management

---

## Part 13: Hands-on - Prepare dev/test Two Sets of Values for Current Helm Chart

If you've already done the Helm Chart in Part 8, this step is ideal to follow up.

Enter the Helm Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

### Step 1: Create values-dev.yaml

Execute:

    cat > values-dev.yaml <<'EOF'
    replicaCount: 1

    image:
      repository: your Harbor domain/test/manual-web
      tag: "dev-13"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    containerPort: 80

    namespace: dev
    EOF

### Step 2: Create values-test.yaml

Execute:

    cat > values-test.yaml <<'EOF'
    replicaCount: 2

    image:
      repository: your Harbor domain/test/manual-web
      tag: "test-13"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    containerPort: 80

    namespace: test
    EOF

### Current Step Understanding

At this point, the multi-environment approach starts to become clearer:

- The structure remains the same Chart
- Differences are expressed through values

---

## Part 14: Hands-on - Render dev/test Two Results with Helm

### Render dev

Execute:

    helm template manual-web . -f values-dev.yaml -n dev

### Render test

Execute:

    helm template manual-web . -f values-test.yaml -n test

### What to Focus On

You mainly look at:

- namespace
- image tag
- replicas

### Current Step Understanding

You'll see:

- The template itself hasn't changed
- But the rendered results differ due to different values

This is the core of multi-environment management.

---

## Part 15: Recommended Multi-Environment Approach at This Stage

Combined with your current learning pace, it's recommended to fix the following approach first.

### Rule 1: Use namespace for Environment Isolation First

Simplest and most intuitive.

### Rule 2: Only Split the Most Critical Differences First

At this stage, at least split:

- namespace
- image tag
- replicas

### Rule 3: Image tag Should Carry Environment or Source Semantics

For example:

- `dev-13`
- `test-13`
- `dev-a1b2c3d-1301`

### Rule 4: Gradually hand over to Helm values as resources increase

Don't manually maintain too much duplicated YAML from the start.

---

## Part 16: Intentionally Do a Small Exercise on "Environment Mixing"

This exercise is important because it will make you clearer why environments need to be split.

### Scenario

Accidentally update the test environment to the dev tag, for example:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain name/test/manual-web:dev-13

Then verify the test environment content:

    kubectl -n test rollout status deployment/manual-web

Enter the cluster to verify:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding to be established now

You will see:

- The test environment is actually running dev content

This is one of the most typical problems when environment boundaries are mixed:

**Once environment boundaries are mixed, the semantics of testing and deployment become chaotic.**

---

## Part 17: This Chapter's Exercise

### Exercise 1: Set up dev / test two minimal environments yourself

Requirements:

- Two namespaces
- Two different tags
- Two different replica counts

### Exercise 2: Do another version of dev-14 and test-14

Requirements:

- Build / push separately
- Update dev and test separately
- Verify that the two environments do not affect each other

### Exercise 3: Answer the following 5 questions yourself

1. Why do we need to split dev, test, prod
2. What are the most common differences in multi-environment setups
3. Which differences are most suitable to split first at this stage
4. Why Helm values are particularly suitable for multi-environment management
5. What are the most common risks of environment mixing

If you can explain these 5 questions yourself, you've mastered this chapter.

---

## Content to be able to explain after this chapter

After completing this chapter, it's recommended to be able to explain the following passage clearly:

The essence of multi-environment deployment is not maintaining three completely different systems, but allowing the same application to use different parameter combinations in different targets.  
The most common and suitable differences to split first at this stage include namespace, image tag, and replicas.  
In this experiment, you can first simulate two environments using `dev` and `test` two namespaces, then differentiate them using different image tags and replica counts.  
As resources increase, manually maintaining multiple YAML files will become increasingly difficult, at which point Helm values are particularly suitable for managing multi-environment parameters.

## Common Issues and Troubleshooting Directions

### Issue 1: Why do the YAMLs for two environments look almost the same and still need to be separated

Because their targets are different.  
Even if only the image tag and replica count differ, it's already enough to constitute environment differences.

### Issue 2: Why is a dev environment necessary now that there's only test

It's necessary.  
Even just to establish an environment isolation mindset, it's worth setting it up first.

### Issue 3: If prod hasn't been set up yet, is this chapter meaningless

No.  
Currently, the most important is to first establish:

- Environments need to be split
- Differences need to be converged
- Parameters need to be manageable

Prod is just a subsequent extension along this line of thinking.

---

## Key Points of This Chapter

1. Why dev / test / prod need to be split
2. The most common differences in multi-environment setups
3. The three key differences most suitable to split first at this stage
4. How to simulate dev / test in the current cluster
5. How Helm values manage multi-environment parameters

## One-Sentence Summary

The core of multi-environment deployment is not replicating multiple systems, but allowing the same application template to use different parameter combinations in different targets. At this stage, the most worthwhile differences to clarify are namespace, image tag, and replicas.