# 20-K8s Release Strategy Advanced: Blue-Green, Canary, and Gray Release Introduction

## Document Notes

This is the 20th note in the 08-CI-CD learning path.

Previously, notes 01-19 have established a minimal release pipeline and gradually filled in:

- Manual release
- Image building and tag
- Harbor repository
- GitLab CI / Jenkins Pipeline
- Deployment rolling update
- Helm
- Multi-environment release
- Executor
- Security basics
- Harbor advanced
- GitOps basics

This article begins to explore the next level of release strategies:

**Why do enterprises mention blue-green, canary, and gray releases in addition to the common rolling update?**

This article does not require you to set up Service Mesh, Ingress Controller, and traffic gateways today.  
Current focus is:

1. Clearly explain the goals and boundaries of the three strategies
2. Perform a "practical simplified experiment" using your existing minimal environment
3. Establish a basic judgment of "when to use which strategy"

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Existing minimal application `manual-web`

## Tags

#Kubernetes #CI-CD #BlueGreenLaunch #CanaryRelease. #GreyscaleRelease #Deployment #Service #PublishingPolicy #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. What problems each strategy (rolling update, blue-green, canary, gray) solves
2. The core difference between blue-green and rolling update
3. Why canary and gray are often mentioned together
4. Perform a minimal blue-green switch experiment in the current cluster
5. Perform a minimal "canary ratio release" simulation experiment in the current cluster
6. Explain the applicable scenarios of the three strategies

## This Article's Experiment Mainline

This article is divided into 4 sections:

1. Compare the goals of rolling update, blue-green, canary, and gray
2. Perform a minimal blue-green release experiment in the current environment
3. Perform a minimal "canary ratio release" simulation experiment in the current environment
4. Summarize when to use each strategy

---

## Part 1: First, Review What Rolling Update Actually Does

You've already done this multiple times:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:vX

Then observe:

- New ReplicaSet appears
- New Pod starts
- Old Pod exits gradually

This is the most typical rolling update.

### Core Characteristics of Rolling Update

- Only maintain one Service entry
- Continuous update within the same Deployment
- New and old Pods coexist for a period
- Final new Pod replaces old Pod

### Problems Solved by Rolling Update

- Basic smooth release
- Avoid full stop and restart
- Suitable for most ordinary stateless services

### Understanding to Establish at This Stage

Rolling update is the default, most common, and most basic release strategy.  
The blue-green, canary, and gray strategies that follow are not negations of it, but provide finer control for more complex requirements.

---

## Part 2: What is Blue-Green Release, How to Understand It

Blue-green release can be initially understood as:

**Prepare two complete versions simultaneously, only let one of them actually receive traffic, and switch traffic at a certain moment.**

You can simply think of it as:

- Blue: current old version
- Green: new version

During release:

1. Old version continues to provide service
2. New version is fully started but does not receive formal traffic or not as the main traffic entry
3. After confirming the new version is fine
4. Switch traffic to the new version through Service or entry

### Problems Solved by Blue-Green Release

- Clear switch action
- Faster rollback
- Easier to do "full switch"
- More suitable for scenarios with high requirements for switch controllability

### Understanding to Establish at This Stage

The core of blue-green release is not "gradually replacing Pods",  
but:

**Prepare two versions in advance, then switch traffic.**

---

## Part 3: What is Canary Release, How to Understand It

Canary release can be initially understood as:

**Let a small amount of traffic enter the new version first, observe if there are issues, then gradually expand.**

The most intuitive understanding is:

- Old version first carries most traffic
- New version only carries a small portion of traffic
- Expand proportion after observing stability

### Problems Solved by Canary Release

- Reduce the risk of full-scale switch
- Let small traffic test run first
- More suitable for scenarios where "small-scale real effect observation" is needed

### Understanding to Establish at This Stage

The key term for canary is:

**Small proportion trial release.**

---

## Part 4: What is Gray Release, How to Understand It

The term "gray release" is often mixed with canary in many teams, but let's make a practical distinction at this stage.

### Practical Understanding at This Stage

#### Canary Release

Emphasizes more:

- Small proportion of traffic enters the new version first

#### Gray Release

Emphasizes more:

- Phased, ranged, and gradually expanding impact

Gray can be based on:

- User range
- Region
- Instance ratio
- Traffic ratio
- Certain specific groups or conditions

### Understanding to Establish at This Stage

At this stage, you can first understand the relationship as:

- Canary is a typical implementation of gray
- Gray is a broader concept of "gradual scaling"

---

## Part 5: Comparison of the Three Strategies with Rolling Update

### 1) Rolling Update

Core action:

- Gradually replace within the same Deployment

Advantages:

- Simple
- Native support
- Easiest to get started

Disadvantages:

- Traffic switching is not independent
- New and old versions are more tightly bound during switching

### 2) Blue-Green Release

Core action:

- Two versions coexist
- Switch traffic through entry or Service

Advantages:

- Clear switching
- Fast rollback
- Easier to do full comparison

Disadvantages:

- Higher resource consumption
- Maintain two versions simultaneously

### 3) Canary / Gray

Core action:

- Small traffic enters the new version first
- Expand after observation

Advantages:

- Risk control at a finer granularity
- More suitable for real traffic testing

Disadvantages:

- Higher implementation complexity
- Higher traffic control requirements

### Understanding to Establish at This Stage

These three are not mutually exclusive, but choices under different levels of complexity and risk control objectives.

---

## Part 6: Hands-on - Perform a Minimal Blue-Green Release Experiment

This section uses your existing `manual-web` to perform a minimal blue-green switch.

### Experiment Plan

We won't directly perform a rolling update on the same Deployment, but prepare two Deployments:

- `manual-web-blue`
- `manual-web-green`

Then only let one Service select one of them.

---

## Part 7: Prepare the Blue Version

### Step 1: Prepare Blue Content

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Write the blue page: /think

cat > index.html <<'EOF'
<html>
  <head>
    <meta charset="utf-8">
    <title>manual web blue</title>
  </head>
  <body>
    <h1>Hello K8s CI/CD</h1>
    <p>version: blue</p>
  </body>
</html>
EOF

### Step 2: Build and Push blue Image

    docker build -t manual-web:blue .
    docker tag manual-web:blue your Harbor domain name/test/manual-web:blue
    docker push your Harbor domain name/test/manual-web:blue

---

## Part 8: Prepare green Version

### Step 1: Prepare green Content

Modify the page:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual web green</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: green</p>
      </body>
    </html>
    EOF

### Step 2: Build and Push green Image

    docker build -t manual-web:green .
    docker tag manual-web:green your Harbor domain name/test/manual-web:green
    docker push your Harbor domain name/test/manual-web:green

---

## Part 9: Create blue and green Deployments

### Step 1: Create blue/green YAML

Execute:

    cat > blue-green.yaml <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: manual-web-blue
      namespace: test
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: manual-web
          version: blue
      template:
        metadata:
          labels:
            app: manual-web
            version: blue
        spec:
          containers:
            - name: manual-web
              image: your Harbor domain name/test/manual-web:blue
              imagePullPolicy: IfNotPresent
              ports:
                - container
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: manual-web-green
      namespace: test
    spec:
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
              image: your Harbor domain name/test/manual-web:green
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

Replace `你的Harbor域名` with actual values.

### Step 2: Apply Configuration

    kubectl apply -f blue-green.yaml

### Step 3: Check Resources

    kubectl -n test get deploy,pods,svc

### Understanding at This Step

At this point:

- Both blue and green versions are running
- But the Service currently selects blue

This is the critical prerequisite for blue-green deployment.

---

## Part 10: Verify blue is Receiving Traffic

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web-bluegreen.test.svc.cluster.local

### Expected Outcome

You should see:

    version: blue

### Understanding at This Step

The green version is running but hasn't become the active backend for the Service yet.

---

## Part 11: Switch to green, Complete Minimal Blue-Green Traffic Switch

### Step 1: Modify Service Selector

Execute: /think

kubectl -n test patch svc manual-web-bluegreen -p '{"spec":{"selector":{"app":"manual-web","version":"green"}}}'

### Step 2: Verify Again by Accessing

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web-bluegreen.test.svc.cluster.local

### Expected Phenomenon

You should see:

    version: green

### Understanding This Step

No Deployment rolling update has occurred here,  
nor has "new Pod gradually replacing old Pod" happened.

Instead:

- Both versions are already running
- The Service's traffic entry has simply switched from blue to green

This is the core action of blue-green deployment.

---

## Part 12: Perform a Minimal Rollback, Experience Blue-Green Advantages

### Step 1: Switch the Service Back to blue

Execute:

    kubectl -n test patch svc manual-web-bluegreen -p '{"spec":{"selector":{"app":"manual-web","version":"blue"}}}'

### Step 2: Verify Again by Accessing

Execute the cluster verification command.

### Understanding This Step

You'll find that rolling back blue-green deployment is very straightforward:

- No need to rebuild
- No need to re-rollout
- No need to start a new version

Just:

**Switch the traffic entry back to the old version.**

This is one of the typical advantages of blue-green deployment.

---

## Part 13: Hands-on - Simulate a Minimal Canary Release

At this stage, no Ingress Controller, Service Mesh, or traffic gateway is integrated,  
so we use a "minimum observable simulation method" to understand canary.

### Core Idea

Use the same Service to select both:

- Old version Pod
- New version Pod

Then simulate traffic ratio through "replica ratio".

This is not precise production-level traffic control, but sufficient to help you understand canary's core.

---

## Part 14: Prepare stable and canary versions

### Step 1: Prepare stable image

Write stable content:

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

Build and push:

    docker build -t manual-web:stable .
    docker tag manual-web:stable your Harbor domain/test/manual-web:stable
    docker push your Harbor domain/test/manual-web:stable

### Step 2: Prepare canary image

Modify content:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual web canary</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: canary</p>
      </body>
    </html>
    EOF

Build and push:

    docker build -t manual-web:canary .
    docker tag manual-web:canary your Harbor domain/test/manual-web:canary
    docker push your Harbor domain/test/manual-web:canary

---

## Part 15: Create stable / canary two Deployment, and let Service select both

### Step 1: Create YAML

Execute: /think

```yaml
cat > canary.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: manual-web-stable
  namespace: test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: manual-web-canary
      track: stable
  template:
    metadata:
      labels:
        app: manual-web-canary
        track: stable
    spec:
      containers:
        - name: manual-web
          image: Yours.HarborDomain Name/test/manual-web:stable
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: manual-web-canary
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: manual-web-canary
      track: canary
  template:
    metadata:
      labels:
        app: manual-web-canary
        track: canary
    spec:
      containers:
        - name: manual-web
          image: Yours.HarborDomain Name/test/manual-web:canary
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: manual-web-canary-svc
  namespace: test
spec:
  selector:
    app: manual-web-canary
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

### Step 2: Apply Configuration

```bash
kubectl apply -f canary.yaml
```

### Step 3: Check Pods

```bash
kubectl -n test get pods -l app=manual-web-canary -o wide
```

### Current Understanding to Establish

At this point, the Service will select:

- 3 stable Pods
- 1 canary Pod

In the coarsest understanding, you can think of it as:

- Approximately 75% to stable
- Approximately 25% to canary

This is not a strict production-grade traffic control, but it's sufficient to understand the "small amount of traffic enters the new version" concept of canary.

---

## Part 16: Access Multiple Times, Observe Possible Different Results

Execute:

```bash
kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh
```

After entering, execute multiple times:

```bash
wget -qO- http://manual-web-canary-svc.test.svc.cluster.local
```

Repeating multiple times, you may see:

- `version: stable`
- `version: canary`

Alternating.

### Current Understanding to Establish

This is the minimal canary simulation:

- The Service distributes traffic to both the old and new versions simultaneously
- The replica ratio roughly influences the traffic distribution ratio

---

## Part 17: Expand Canary Ratio, Simulate "Gray Deployment"

### Step 1: Scale canary replicas from 1 to 2

Execute:

```bash
kubectl -n test scale deploy manual-web-canary --replicas=2
```

### Step 2: Check Pod Count Again

```bash
kubectl -n test get pods -l app=manual-web-canary
```

### Step 3: Access Again Multiple Times

Repeat executing:

```bash
wget -qO- http://manual-web-canary-svc.test.svc.cluster.local
```

### Current Understanding to Establish

Now the ratio is approximately:

- stable 3
- canary 2

That is, the new version's proportion has increased.

This reflects the "gray deployment" approach:

- Start with a small amount
- Observe
- Then gradually expand

---

## Part 18: When to Use Which Strategy

This section is very important, and you must establish the minimal judgment.

### Suitable Scenarios for Rolling Update

- Ordinary stateless services
- Relatively controllable risks
- Not sensitive to additional resource consumption
- Doesn't require complex traffic governance

### Suitable Scenarios for Blue-Green Deployment

- Want a very clear switch action
- Want fast rollback
- Can accept the resource consumption of running two versions simultaneously
- Prioritize "switch control" over gradual rollout

### Suitable Scenarios for Canary / Gray Deployment

- Want to test with small traffic
- Want to observe real user effects first
- Require finer risk control
- Capable of doing more granular traffic management

### Current Understanding to Establish

Not all systems must use blue-green or canary.  
Often:

- Ordinary business can use rolling update alone
- High-risk releases are more worth adopting more complex strategies

---

## Part 19: How to Remember These Three Strategies at This Stage

It's recommended to fix the following terminology first.

### Rolling Update

- Replace Pods internally within the same Deployment

### Blue-Green Deployment

- Two versions run simultaneously
- Switch traffic through the entry point

### Canary Release

- A small amount of traffic enters the new version first

### Gray Deployment

- Gradually expand the impact scope
- Canary is one of the typical implementation methods

---

## Part 20: This Article's Mini Exercise

### Exercise 1: Independently Perform a Blue-Green Switch Again

Requirement: /think

- Build your own `blue2` / `green2`
- Deploy yourself
- Switch Service selector yourself
- Verify page content yourself
- Switch back

### Exercise 2: Redo the Canary Ratio Experiment Independently

Requirements:

- stable 3 replicas
- canary 1 replica
- Observe results after multiple accesses
- Expand canary to 2 replicas
- Observe results again

### Exercise 3: Answer the Following 5 Questions Yourself

1. What is the core difference between rolling update and blue-green deployment?
2. Why is rollback faster in blue-green deployment?
3. What is the core idea of canary deployment?
4. What's the relationship between gray release and canary deployment?
5. Why is the canary in this experiment just a "minimal simulation," not equivalent to full production-grade traffic governance?

If you can explain these 5 questions yourself, you've mastered this article.

---

## Content to Be Able to Explain After This Section

After completing this section, it's recommended to be able to explain the following passage yourself:

Kubernetes' most common default deployment method is rolling update, which completes version switching by gradually replacing old and new Pods.  
Blue-green deployment prepares two complete versions at the same time, switching traffic from the old version to the new version through Service or entry points, making rollback more direct.  
The core of canary deployment is to let a small amount of traffic enter the new version first, then gradually expand after confirming stability. Gray release can be understood as a broader concept of gradual scaling.  
In the current experimental environment, you can understand blue-green deployment through two Deployment setups and Service selector switching, and also simulate a minimal canary ratio by having stable/canary with different replica counts simultaneously access a Service.

## Common Issues and Troubleshooting Directions

### Issue 1: Why does blue-green deployment seem like "running an extra Deployment"?

Because it inherently relies on two versions running in parallel. The core isn't gradual replacement but traffic switching.

### Issue 2: Why does the canary experiment sometimes show stable and sometimes canary?

Because the Service distributes requests to different Pods. You're currently using replica ratios for a minimal simulation.

### Issue 3: Is the canary in this experiment a production-grade practice?

No.  
This is just helping you understand the idea of "letting a small amount of traffic enter the new version first."  
More precise ratio control usually requires Ingress Controller, Service Mesh, or more specialized traffic gateways.

---

## Key Points to Master in This Section

1. Differences in goals between rolling update, blue-green, canary, and gray releases
2. Minimum traffic switching experiment for blue-green deployment
3. Minimum canary ratio simulation experiment
4. Which scenarios are more suitable for which strategies
5. The boundary between current experiments and production-grade implementations

## One-Sentence Summary

Rolling update solves the problem of basic smooth deployment, blue-green deployment solves the issue of overall switching between two versions and quick rollback, while canary and gray release address risk control for small-scale traffic testing and gradual scaling.