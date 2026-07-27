# Advanced 20-K8s Release Strategies: Introduction to Blue-Green, Canary, and GrayRelease

## Document Description

This article is the 20th note in the 08-CI-CD learning pathway.

Previously, in articles 01-19, we established a basic release pipeline, covering:

- Manual releases
- Image building and tagging
- Harbor repository management
- GitLab CI/Jenkins pipelines
- Deployment rolling updates
- Helm usage
- Multi-environment deployments
- Execution frameworks
- Security fundamentals
- Advanced Harbor configurations
- GitOps basics

In this article, we delve deeper into release strategies:

**Why do companies also use Blue-Green, Canary, and GrayRelease in addition to the common rolling updates?**

This article doesn’t require you to set up Service Mesh, Ingress Controller, or traffic gateways today.  
The focus here is on:

1. Clearly understanding the goals and boundaries of each strategy
2. Conducting practical experiments using your existing minimal environment
3. Developing a basic judgment about when to use which strategy

This article continues to use the current setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Private Harbor repository
- `test` namespace
- Existing minimal application `manual-web`

## Tags

#Kubernetes #CI-CD #Blue-Green Release #Canary Release #GrayRelease #Deployment #Service #Release Strategies #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand the problems each of rolling updates, Blue-Green, Canary, and GrayRelease aims to solve
2. Distinguish between Blue-Green releases and rolling updates
3. Recognize why Canary and GrayRelease are often mentioned together
4. Conduct a minimal Blue-Green switch experiment in your current cluster
5. Perform a minimal “Canary proportion release” simulation experiment in your current cluster
6. Clearly explain the applicable scenarios for each strategy

## Experimental Outline of This Article

This article is divided into 4 sections:

1. Compare the goals of rolling updates, Blue-Green, Canary, and GrayRelease
2. Conduct a minimal Blue-Green release experiment in your current environment
3. Perform a minimal Canary proportion release experiment in your current environment
4. Summarize when to use which strategy

---

## Section 1: Review of Rolling Updates

You have already performed this multiple times:

    kubectl -n test set image deployment/manual-web manual-web=yourHarbor域名/test/manual-web:vX

Then observe:

- A new ReplicaSet is created
- New Pods start running
- Old Pods gradually shut down

This is the typical example of a rolling update.

### Key Features of Rolling Updates

- Only one Service entry point is maintained
- The same Deployment is continuously updated
- Old and new Pods coexist for a period of time
- Eventually, new Pods replace old Pods

### Problems Solved by Rolling Updates

- Provides smooth basic releases
- Avoids complete system shutdowns
- Suitable for most common stateless services

### Key Points to Understand

Rolling updates are the default, most common, and fundamental release strategy.  
Blue-Green, Canary, and GrayRelease do not replace them but offer more detailed control in more complex scenarios.

---

## Section 2: Understanding Blue-Green Releases

Blue-Green releases can be understood as:

**Preparing two complete versions simultaneously, with only one version receiving actual traffic, and then switching the traffic at a specific moment.**

You can think of it simply as:

- Blue: The current old version
- Green: The new version

During release:

1. The old version continues to provide services
2. The new version is fully set up but not yet used for official traffic
3. Once the new version is confirmed to be stable:
4. The traffic is switched entirely to the new version through changes in Service or entry points

### Problems Solved by Blue-Green Releases

- Clearer switching process
- Faster rollback capabilities
- Easier to perform complete comparisons
- More suitable for scenarios requiring high control over the switch

## Key Points to Understand

The core of Blue-Green releases is not **gradual Pod replacement**,  
but rather **preparing two versions in advance and then switching traffic**.

---

## Section 3: Understanding Canary Releases

Canary releases can be understood as:

**First allowing a small amount of traffic to enter the new version, observing for any issues, and then gradually increasing the proportion.**

The most straightforward way to understand this is:

- The old version handles most of the traffic initially
- Only a small portion of traffic goes through the new version
- The proportion is increased after observing stability

### Problems Solved by Canary Releases

- Reduces the risk associated with sudden full-scale switches
- Allows for testing in a limited environment first
replicas: 2
selector:
  matchLabels:
    app: manual-web
    version: green
template:
  metadata:
    labels:
      app: manual-web
      version: green
spec:
  containers:
  - name: manual-web
    image: your-Harbor-domain/test/manual-web:green
    imagePullPolicy: IfNotPresent
    ports:
      - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: manual-web-bluegreen
  namespace: test
spec:
  selector:
    app: manual-web
    version: blue
ports:
  - protocol: TCP
    port: 80
    targetPort: 80
type: ClusterIP
EOF

Replace `your-Harbor-domain` with the actual value.

### Step 2: Apply the Configuration

Run the following command:

kubectl apply -f blue-green.yaml

### Step 3: Check the Resources

Use the following command to view the deployed resources:

kubectl -n test get deploy,pods,svc

### Understanding This Step

At this point:

- Both the blue and green versions are running.
- However, the Service is currently set to use the blue version.

This is a crucial prerequisite for implementing blue-green deployment.

---

## Section 10: Verify That the Blue Version Is Receiving Traffic

Execute the following command:

kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the command, execute:

wget -qO- http://manual-web-bluegreen.test.svc.cluster.local

### Expected Outcome

You should see:

version: blue

### Understanding This Step

Although the green version is already running, it is not currently serving traffic for the Service.

---

## Section 11: Switch to the Green Version to Complete Minimum Blue-Green Traffic Switching

### Step 1: Modify the Service Selector

Run the following command:

kubectl -n test patch svc manual-web-bluegreen -p '{"spec":{"selector":{"app":"manual-web","version":"green"}}}'

### Step 2: Verify Again

Run the following commands to check the changes:

kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the command, execute:

wget -qO- http://manual-web-bluegreen.test.svc.cluster.local

### Expected Outcome

You should see:

version: green

### Understanding This Step

In this case, there is no need for a Deployment rollout or for new Pods to replace old ones. Instead, the Service simply switches its traffic from the blue version to the green version.

This is the core mechanism of blue-green deployment.

---

## Section 12: Perform a Minimum Rollback to Experience the Advantages of Blue-Green Deployment

### Step 1: Switch the Service Back to the Blue Version

Run the following command:

kubectl -n test patch svc manual-web-bluegreen -p '{"spec":{"selector":{"app":"manual-web","version":"blue"}}}'

### Step 2: Verify Again

Run the commands from Step 2 again to check if the service has switched back to the blue version.

### Understanding This Step

You will see that rolling back a blue-green deployment is very straightforward. There is no need for rebuilding, rolling out, or starting a new version. Simply switching the traffic back to the old version is enough.

This is one of the key advantages of blue-green deployment.

---

## Section 13: Practical Example – Simulate a Minimum Canary Release

At this stage, we will not use an Ingress Controller, Service Mesh, or traffic gateway. Instead, we will use a "minimum observable simulation method" to understand the concept of canary releases.

### Core Idea

Use the same Service to select both:

- Old version Pods
- New version Pods

Then, use the "number of replicas" to roughly simulate the traffic distribution between the two versions.

This is not precise production-level traffic control, but it is sufficient to help you understand the basic principles of canary releases.

---

## Section 14: Prepare the Old Stable Version and the New Canary Version

### Step 1: Prepare the Stable Image

Write the content for the stable version:

cd ~/08-ci-cd/01-manual-release

cat > index.html <<'EOF'
<html>
  <head>
    <meta charset="utf-8">
    <title>manual web stable</title>
  </head>
  <body>
    <h1>Hello K8s CI/CD</h1>
    <p>version: stable</p>
  </body>
</html>
EOF

Build and push the image:

docker build -t manual-web:stable .
docker tag manual-web:stable your-Harbor-domain/test/manual-web:stable
docker push your-Harbor-domain/test/manual-web:stable

### Step 2### Summary

This chapter explores various deployment strategies in Kubernetes, focusing on rolling updates, blue-green releases, canary deployments, and grayscale approaches. Rolling updates involve gradual replacement of Pods within the same Deployment to manage version transitions smoothly. Blue-green releases prepare two complete versions simultaneously and switch traffic between them through Service or entry points, allowing for quick rollbacks. Canary deployments introduce a small amount of traffic to the new version first, enabling observation before scaling it up. Grayscale releases gradually increase the impact of the new version, providing a more refined control over traffic distribution.

Each strategy has its appropriate use cases: rolling updates are suitable for general stateless services with manageable risks; blue-green releases are ideal for scenarios requiring clear switching and rapid rollback; canary and grayscale deployments are better suited for testing new versions in small volumes and observing real-user behavior. Understanding the differences between these strategies and when to use them is crucial for effective Kubernetes management.