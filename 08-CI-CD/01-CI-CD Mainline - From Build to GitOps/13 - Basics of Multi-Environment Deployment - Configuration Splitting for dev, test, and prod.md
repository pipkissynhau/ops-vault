# 13 - Basics of Multi-Environment Deployment: Configuration Splitting for dev, test, and prod

## Document Description

This article is the 13th note in the 08-CI-CD learning pathway.

In previous sections 01-12, we have set up a minimal deployment pipeline and established mechanisms for image tagging and image retrieval.  
This article builds on that by addressing a very practical issue in a deployment system:

**Why must we start distinguishing between dev, test, and prod environments once team collaboration and continuous deployment begin?**

Many people starting to learn about K8s or CI/CD initially work with just one environment, where everything is combined.  
However, as soon as tasks like development validation, integration testing, stable deployment, and official launch emerge, it becomes necessary to separate these environments.

The focus of this article is not on setting up all three environments at once but on establishing a **practical, maintainable approach suitable for the current stage**.

This article continues to use the following current environment setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The existing `test` namespace
- The existing minimal application `manual-web`

## Tags

#Kubernetes #CI-CD #Multi-Environment Deployment #dev #test #prod #Helm #values #Namespace #Configuration Splitting #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand why it is essential to separate dev, test, and prod environments.
2. Identify the most common differences between these environments.
3. Determine which differences are appropriate to address at this stage and which should not be over-designed.
4. Use namespaces in your current cluster to simulate dev and test environments.
5. Apply different image tags and replica counts for these two environments.
6. Comprehend why Helm values are particularly suitable for managing multi-environment settings.
7. Clearly explain that "multi-environments do not mean multiple systems but rather different parameter combinations for the same application under various goals."

## Main Experimental Approach of This Article

This article is divided into 4 sections:

1. Establish a basic understanding of multi-environments.
2. Simulate dev and test environments in the current cluster.
3. Set up two deployment scenarios using different tags and replica counts.
4. Organize environment differences using Helm values.

---

## Section 1: Why Separate dev, test, and prod Environments

Let's start with practical issues rather than considering a comprehensive platform.

Suppose you currently have only one `test` environment and the following requirements arise:

- Developers want to quickly verify new changes.
- Testers need a relatively stable version for testing.
- You want to retain a stable configuration that is closer to an official release.

If these requirements are combined in one environment, several problems will immediately emerge:

### Problem 1: Conflicting Goals

- Developers want fast iteration.
- Testers need stability.
- Pre-launch verification requires traceability.

A single environment cannot meet all these needs simultaneously.

### Problem 2: Unclear Version Labels

It becomes difficult to distinguish whether a deployment is a "temporary development version," a "testing version," or a "pre-production version."

### Problem 3: Different Configuration Requirements

Different environments often have different requirements for:

- Replica counts
- Image tags
- Access addresses
- Resource limits
- Dependency configurations.

Combining them all will lead to increased complexity.

### Understanding to Establish at This Stage

Separating dev, test, and prod environments is not about making the system more complicated but about:

- Clarifying goals.
- Making versions more controllable.
- Making configurations more manageable.

---

## Section 2: Understanding the Focus of Each Environment in One Sentence

For now, let's establish a practical understanding without getting too technical.

### dev

Focus on development validation:

- Fast updates.
- Frequent changes allowed.
- High fault tolerance.
- No long-term stability required.

### test

Focus on testing and integration:

- Requires a certain level of stability.
- Must be repeatable for testing purposes.
- Versions should be clear and distinct.

### prod

Focus on official services:

- Stability is the top priority.
- Changes must be made carefully.
- Versions must be traceable.
- Rollbacks must be well-defined.

### Understanding to Establish at This Stage

These three environments are not "technically different systems" but rather:

**The same application with different versions and parameters for different purposes.**

---

## Section 3: The Most Common Differences Between Environments

For now, we don't need to identify all differences; let's focus on the 5 most common ones.

### 1) Image Tags

For example:

- dev: Can be updated more frequently, geared towards development.
- test: Likely a testing version with specific tags.
- prod: An official or release version with stable tags.

### 2) Replicametadata:
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

### Understanding Required for This Step

Here, we use the principle of minimum difference to simulate the dev environment:

- namespace = dev
- replicas = 1
- image = dev-13

This is a very typical configuration for a “development and validation environment”.

---

## Section 8: Practical Operations – Preparing a Minimum YAML for the Test Environment

### Step 1: Create the test environment YAML

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
              image: yourHarbor域名/test/manual-web:test-13
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

### Understanding Required for This Step

Here, we use the principle of minimum difference to simulate the test environment:

- namespace = test
- replicas = 2
- image = test-13

By doing this, you begin to understand how “the same application can have different target parameters in different environments”.

---

## Section 9: Practical Operations – Deploying the dev and Test Environments Respectively

### Step 1: Deploy the dev environment

Execute:

    kubectl apply -f manual-web-dev.yaml

Check results:

    kubectl -n dev get deploy,pods,svc

### Step 2: Deploy the test environment

Execute:

    kubectl apply -f manual-web-test.yaml

Check results:

    kubectl -n test get deploy,pods,svc

### Understanding Required for This Step

By now, you have two separate environments within the same cluster:

- dev / manual-web
- test / manual-web

They:

- Can have the same name
- Share the same resource logic
- But are separated by namespace and image tag

This is the most basic practice of deploying in multiple environments.

---

## Section 10: Practical Operations – Verifying the Business Results of the dev and Test Environments Respectively

### Verify the dev environment

Execute:

    kubectl -n dev run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the script, execute:

    wget -qO- http://manual-web.dev.svc.cluster.local

You should see:

    version: dev-13

### Verify the test environment

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the script, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

You should see:

    version: test-13

### Understanding Required for This Step

This step is very important because it demonstrates that:

- The same application name
- Can exist in different namespaces
- And can truly support different environmental versions simultaneously

This is the most direct and essential experience of managing multiple environments.

---

## Section 11: Summarizing the Current Minimum Differences Between dev and test

You have now created two separate environments. It’s time to summarize the current differences:

### Current differences in the dev environment

- namespace: `dev`
- replicas: 1
- image tag: `dev-13`

### Current differences in the test environment

- namespace: `test`
- replicas: 2
- image tag: `test-13`

### Understanding Required for This Step

This represents the minimum feasible separation of multiple environments at this stage:

- It’s not necessary to separate all parameters from the beginning;
- Identifying and separating the most critical differences is sufficient.

---

## Section 12: Why Helm Values Are Particularly Suitable for Managing Multiple Environments

Now that you have completed these steps, looking back at Helm makes it much clearer.

Without using Helm, you would need to maintain two separate YAML files:

- `manual-web-dev.yaml`
- `manual-web-test.yaml`

As the number of files increases, you will quickly encounter issues such as:

- A lot of duplicate content
- The need to modify both files when making structural```bash
kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-13

Then verify the content of the test environment:

kubectl -n test rollout status deployment/manual-web

Enter the cluster to verify further:

kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Execute:

wget -qO- http://manual-web.test.svc.cluster.local

### Key Points to Understand

You will observe that:

- The test environment is actually using development settings.

This illustrates one of the most common issues when environments are not clearly defined:

**When the boundaries between environments blur, the semantics of testing and deployment become confused.**

---

## Section 17: Practice Exercises

### Exercise 1: Set up your own minimal dev/test environments

Requirements:

- Two separate namespaces
- Different tag values for each environment
- Varying numbers of replicas

### Exercise 2: Create versions named dev-14 and test-14

Requirements:

- Build and push each version separately
- Update the dev and test environments accordingly
- Verify that they do not affect each other

### Exercise 3: Answer these 5 questions yourself

1. Why is it necessary to separate dev, test, and prod environments?
2. What are the most common differences between multiple environments?
3. At what stage is it most appropriate to start separating these environments?
4. Why are Helm values particularly suitable for managing multi-environment settings?
5. What are the most common risks associated with mixing environments?

If you can answer these questions confidently, you have mastered this topic.

---

## Key Points to Discuss After Completing This Section

After studying this section, you should be able to explain clearly the following:

The essence of multi-environment deployment is not about maintaining three completely different systems, but about using the same application with different parameter combinations for various purposes.  
At present, the most common and practical differences to distinguish between environments include namespace, image tag, and number of replicas.  
In this experiment, you can start by using two namespaces, `dev` and `test`, and differentiate them through different image tags and replica counts.  
As resources increase, manually managing multiple YAML files becomes increasingly difficult; at this point, Helm values become an effective tool for managing multi-environment parameters.

## Common Questions and Solutions

### Question 1: If the YAML configurations for two environments look almost identical, why do we need to separate them?

Because their purposes are different. Even slight differences in things like image tags or replica counts are sufficient to create distinct environments.

### Question 2: Since we only have a test environment for now, is it necessary to set up a dev environment?

It is still beneficial. Just establishing the concept of environmental isolation is worth doing at this stage.

### Question 3: We haven’t set up a prod environment yet; does that mean this section is meaningless?

Not at all. The focus here is on establishing the principles of:

- Separating environments
- Clarifying differences between them
- Making parameters manageable

The prod environment can be added later, following these same guidelines.

---

## Key Takeaways

1. Why it is important to separate dev, test, and prod environments.
2. The most common differences between multiple environments.
3. The three key aspects that should be prioritized for separation at this stage.
4. How to simulate dev/test environments in the current cluster setup.
5. How Helm values can help manage multi-environment parameters effectively.

## In One Sentence

The core of multi-environment deployment is to use the same application framework with different parameter settings for various targets. At this early stage, focusing on namespace, image tag, and replicas is the most effective way to start managing these differences.