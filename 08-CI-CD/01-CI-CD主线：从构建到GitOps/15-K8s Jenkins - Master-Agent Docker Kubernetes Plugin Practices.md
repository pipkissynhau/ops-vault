# 15-Jenkins in Kubernetes: Master-Agent, Docker, and Kubernetes Plugin Practices

## Document Notes

This article is the 15th note in the 08-CI-CD learning path.

The previous article has already clarified a key issue:

- GitLab Runner and Jenkins Agent are essentially both pipeline executors
- Pipeline configuration is just the "definition layer"
- The actual environment where build, push, and deploy operations are executed is the executor's environment

This article continues the discussion, focusing on solving this key question:

**If Jenkins itself runs in Kubernetes, how does it execute pipelines?**

This article doesn't require you to fully set up a production-grade Jenkins on K8s system today,  
the focus is on establishing the core structure and minimal implementation path, so you can understand:

- What Jenkins typically looks like in Kubernetes
- The relationship between Master/Controller and Agent
- Why Docker plugin and Kubernetes plugin exist
- Why Jenkins Agents in Kubernetes are often temporary Pods
- How this model corresponds to the Runner/Agent concepts you learned earlier

This article continues to align with the current learning path:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace
- Minimum application build/push/deploy/rollback/Helm deployment experiment already completed

## Tags

#Kubernetes #CI-CD #Jenkins #JenkinsAgent #MasterAgent #KubernetesPlugin #DockerPlugin #WaterlineExecutor #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. The basic architecture of Jenkins in Kubernetes
2. The relationship between Master (or Controller) and Agent
3. Why Jenkins Agents in Kubernetes are often Pods
4. What problems Docker plugin and Kubernetes plugin solve respectively
5. Mapping the "executor" concept learned earlier to the Jenkins on Kubernetes scenario
6. Being able to explain why many enterprises choose to run Jenkins in Kubernetes

## This Article's Experiment Path

This article is divided into 4 sections:

1. First establish the structural understanding of Jenkins on Kubernetes
2. Understand Agent Pod with minimal YAML and minimal Pipeline thinking
3. Understand the responsibility boundaries between Docker plugin and Kubernetes plugin
4. Do a minimal experiment from the "local Agent perspective"

---

## Part 1: First Understand What Jenkins Typically Looks Like in Kubernetes

First establish a minimal structural sense.

When Jenkins runs in Kubernetes, it typically has at least two roles:

### 1) Jenkins Controller (formerly called Master)

It is responsible for:

- Providing Jenkins web interface
- Managing Jobs/Pipelines
- Managing credentials
- Scheduling tasks
- Deciding which Agent executes tasks
- Recording logs and build history

### 2) Jenkins Agent

It is responsible for:

- Actually executing commands in the pipeline
- Pulling code
- Building
- Pushing to Harbor
- Calling kubectl/helm
- Returning logs and results

### Understanding to Establish at This Stage

You can initially understand it as:

- Controller: responsible for "management"
- Agent: responsible for "execution"

This is completely consistent with the executor concept you learned in the previous article.

---

## Part 2: Why Jenkins Doesn't Execute All Commands Directly

This is a critical question.

Theoretically, you could have the Jenkins Controller simultaneously handle:

- Web interface
- Scheduling
- Build
- Push
- Deploy

But this would quickly lead to problems.

### Problem 1: Mixed Responsibilities

If the Controller needs to manage the system and execute build tasks, it will become increasingly heavy as the number of build tasks grows.

### Problem 2: Inflexible Environment

Different pipelines may require different execution environments, such as:

- Some need Maven
- Some need Node.js
- Some need Docker
- Some need kubectl
- Some need Helm

If everything is piled on the Controller, the environment will become increasingly bloated.

### Problem 3: Poor Isolation

A problem in a build task could potentially affect the Jenkins main body.

### Understanding to Establish at This Stage

Therefore, the correct evolution direction for Jenkins is typically:

**Controller handles management, Agent handles execution.**

---

## Part 3: Why Jenkins on Kubernetes Often Uses "Temporary Agent Pods"

This is the most typical place where Jenkins and Kubernetes combine.

In traditional Jenkins scenarios, Agents could be:

- A fixed physical machine
- A fixed virtual machine
- A fixed Docker host

But in Kubernetes, the natural approach is:

**Let Agents appear as temporary Pods.**

### What Are the Benefits?

#### 1) Disposable

Once a pipeline run completes, the Agent Pod can be destroyed.  
No need to maintain many idle execution nodes long-term.

#### 2) More Flexible Environment

Different pipelines can start different image-based Agent Pods, such as:

- Java Agent
- Docker Agent
- kubectl Agent
- Helm Agent

#### 3) More in Line with Kubernetes Scheduling Thinking

The Agent itself is just a "workload," which naturally fits Kubernetes scheduling.

### Understanding to Establish at This Stage

You can first remember this sentence:

> In the Jenkins on Kubernetes scenario, Agents are often not fixed machines but temporary Pods launched to execute pipelines.

---

## Part 4: Mapping the Previously Learned Executor Concept

You previously learned:

- GitLab Runner is the executor for GitLab
- Jenkins Agent is the executor for Jenkins

Now add another layer:

- If Jenkins runs in Kubernetes, Agents often run as Pods

In other words:

### GitLab Side

- GitLab CI defines the process
- Runner handles execution

### Jenkins Side

- Jenkinsfile defines the process
- Agent handles execution
- In Kubernetes, this Agent is often a Pod

### Understanding to Establish at This Stage

At this point, you should be able to explain:

**Jenkins on Kubernetes simply changes the carrier form of the Agent from traditional machines to Kubernetes Pods.**

---

## Part 5: What Is Jenkins' Docker Plugin, and How to Understand It Initially

# In this phase, we won't delve into all plugin features, only focusing on the most practical understanding.

Docker plugin or Docker-related capabilities typically solve the following problems:

- Execute `docker build` in the pipeline
- Execute `docker tag` in the pipeline
- Execute `docker push` in the pipeline
- Enable Jenkins to integrate with container image building

### Understanding to establish at this stage

Docker plugin-related capabilities are more inclined toward:

**"Should the pipeline executor have Docker building capabilities"**

Which leans toward:

- Image building
- Image tagging
- Image pushing

---

## Part 6: What is the Kubernetes plugin for Jenkins, and how to initially understand it

Kubernetes plugin typically solves the following problems:

- How Jenkins dynamically creates Agent Pod
- How Jenkins schedules pipeline tasks to temporary Pods in K8s
- How Jenkins describes the runtime image and environment for an Agent Pod

### Understanding to establish at this stage

Kubernetes plugin leans toward:

**"How Jenkins hands over the executor to Kubernetes for dynamic creation and management"**

Which focuses on:

- Agent lifecycle
- Agent environment definition
- How Agent starts in K8s

---

## Part 7: Differences between Docker plugin and Kubernetes plugin

These two are often confused, so it's crucial to distinguish them.

### Docker-related capabilities lean toward

- How to build images
- How to push images
- How to enable Docker capabilities in the pipeline

### Kubernetes plugin leans toward

- How to start Agent Pod
- How to schedule tasks in Pod
- How to run Jenkins executor in K8s

### Understanding to establish at this stage

You can remember this way:

- Docker-related capabilities: more inclined toward "image building"
- Kubernetes plugin: more inclined toward "executor management"

These are not mutually exclusive; in practice, they often appear together.

---

## Part 8: How a minimal pipeline runs in Jenkins on K8s

Start with a minimal approach to understand.

### Possible execution process

1. Trigger a Pipeline in Jenkins
2. Jenkins Controller reads the Jenkinsfile
3. Controller determines this Pipeline needs an Agent
4. Through the Kubernetes plugin, request K8s to start an Agent Pod
5. After Agent Pod starts, pull the code
6. Agent Pod executes:
   - `docker build`
   - `docker push`
   - `kubectl set image`
   - `kubectl rollout status`
7. After execution, Agent Pod can be destroyed
8. Jenkins Controller records the build result

### Understanding to establish at this stage

Here, Controller acts as the "scheduler",  
Agent Pod is like a "temporary construction team".

---

## Part 9: Why many enterprises put Jenkins into K8s

This point must be understood in the context of reality, not just technical implementation.

### Reason 1: Existing K8s platform capabilities

Since there's already a K8s cluster, it's natural to consider putting more platform components into it for unified management.

### Reason 2: More convenient elastic Agent

Temporary Agent Pod:

- Created when needed
- Destroyed after use
- More flexible than maintaining many fixed build machines

### Reason 3: Easier environment standardization

Define Agent environment via image, rather than manually installing tools on many machines.

### Reason 4: Unified resource scheduling

Jenkins Agent is also scheduled as K8s workloads, making resource management more convenient.

### Understanding to establish at this stage

Jenkins on K8s isn't just "moving Jenkins to another location",  
more importantly:

**Enable elastic executors, image-based environments, and unified resource scheduling.**

---

## Part 10: Minimal YAML perspective to understand Agent Pod

This section doesn't require you to actually connect Jenkins plugins, only to understand from the K8s perspective:

**So-called Jenkins Agent Pod, essentially is just a Pod.**

For example, a minimal conceptual Pod might look like this:

    apiVersion: v1
    kind: Pod
    metadata:
      name: jenkins-agent-demo
      namespace: test
    spec:
      containers:
        - name: agent
          image: alpine:3.20
          command: ["sh", "-c", "sleep 3600"]

### Understanding to establish at this stage

This Pod itself isn't a complete Jenkins Agent,  
but it helps establish an important intuition:

- Agent Pod is first and foremost a Pod
- What image runs in the Pod determines its capabilities
- If the image lacks Docker, kubectl, helm, it can't perform related tasks

In other words:

**Executor capabilities ultimately depend on the content of the container image.**

---

## Part 11: Minimal experiment to simulate "Agent Pod environment"

This section continues with your current known capabilities for minimal simulation, without requiring actual Jenkins integration.

### Step 1: Create a minimal temporary Pod

Execute:

    kubectl -n test run agent-demo --image=alpine:3.20 -- sleep 3600

Check:

    kubectl -n test get pods

### Step 2: Enter the Pod

Execute:

    kubectl -n test exec -it agent-demo -- sh

### Step 3: Observe the environment inside the Pod

Execute:

    uname -a
    ls /
    which docker
    which kubectl
    which helm

### Expected phenomena

You'll likely find:

- The Pod can start
- But it lacks `docker`
- Also lacks `kubectl`
- Also lacks `helm`

### Understanding to establish at this stage

This is why "Agent being just a Pod isn't enough".

**The key isn't just having a Pod, but whether this Pod contains the tools needed for the pipeline.**

### Step 4: Exit and delete the Pod

    exit
    kubectl -n test delete pod agent-demo

---

## Part 12: Another minimal experiment to understand "executor image content"

This section doesn't pursue full functionality, only helping you understand the importance of "executor image content".

Assume you prepare an Agent image in the future, it should at least contain: /think

- shell
- git
- docker or other build tools
- kubectl
- possibly also helm

You don't need to create the image immediately, but first establish a concept:

### Executor Image = Collection of Execution Capabilities

If an Agent image only includes:

- git
- kubectl

It is more suitable for deployment, but not for build images.

If an Agent image includes:

- docker
- git

It is more suitable for build/push, but may not be suitable for helm releases.

### Understanding to Establish at This Stage

Executor image design is actually designing:

**What capabilities should this pipeline task have.**

---

## Part 13: Understanding the Minimum Pipeline Concept for Jenkins on K8s

This section focuses only on structural understanding, not deep syntax exploration.

Assume a Jenkinsfile follows this logic:

    pipeline {
        agent any

        stages {
            stage('Build') {
                steps {
                    sh 'docker build -t manual-web:v15 .'
                }
            }

            stage('Push') {
                steps {
                    sh 'docker push harbor.example.com/test/manual-web:v15'
                }
            }

            stage('Deploy') {
                steps {
                    sh 'kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:v15'
                }
            }
        }
    }

If Jenkins runs on K8s, this pipeline may not execute on the Controller itself, but rather:

- Controller scheduling
- Agent Pod executing the work

### Understanding to Establish at This Stage

Jenkinsfile remains the "definition layer",  
the Agent still performs the actual work,  
but the Agent's hosting method has become a Pod.

---

## Part 14: Why This Section and Section 14 Are Connected

In the previous section, you've learned:

- GitLab Runner / Jenkins Agent is the executor
- Without an executor, the pipeline cannot run

This section continues by adding another layer:

- If Jenkins runs on K8s
- Agents are often dynamically created as Pods
- Docker capabilities and Kubernetes plugin capabilities solve different problems

Thus, these two sections form a pair:

### Section 14 Solves

"What is an executor"

### Section 15 Solves

"How does an executor work if it resides in K8s"

---

## Part 15: This Section's Practice Exercise

### Exercise 1: Draw the Minimum Structure of Jenkins on K8s in Your Own Words

Requirements must include at least:

- Jenkins Controller
- Jenkins Agent
- K8s Cluster
- Harbor
- Deployment

### Exercise 2: Answer the Following 5 Questions Yourself

1. What is the difference between Jenkins Controller and Agent?
2. Why are Agents in K8s often hosted in Pods?
3. What problems do Docker plugin capabilities and Kubernetes plugin capabilities respectively solve?
4. Why does the tools installed in an Agent image determine what it can do?
5. Why is Jenkins on K8s not as simple as "just installing Jenkins elsewhere"?

### Exercise 3: Perform the Temporary Pod Environment Observation Experiment Again

Choose any base image to start a Pod, enter it, and check:

    which git
    which docker
    which kubectl
    which helm

Then summarize:

- What this Pod can do as an executor
- What it cannot do

If you can complete these 3 exercises, you've mastered this section.

---

## Content to Be Able to Explain After This Section

After completing this section, you should be able to explain the following:

The core idea of Jenkins on Kubernetes is to entrust Jenkins' executor capabilities to K8s for dynamic hosting.  
Jenkins Controller manages pipelines and schedules tasks, while Jenkins Agent typically executes build, push, and deploy commands.  
In Kubernetes scenarios, Agents often run as temporary Pods, enabling on-demand creation and destruction, and providing different tool capabilities through various executor images.  
Docker-related capabilities focus more on image building and pushing, while Kubernetes plugins focus on enabling Jenkins to dynamically create Agent Pods and schedule tasks to these Pods.  
Thus, the key focus of Jenkins on K8s is not just Jenkins itself, but how the executor is dynamically managed by K8s.

## Common Issues and Troubleshooting Directions

### Issue 1: Why Does Jenkins on K8s Sound Complex?

Because it's not just Jenkins, but also includes:

- Jenkins Controller / Agent architecture
- K8s scheduling
- Executor image design
- Harbor / kubectl / helm toolchain

### Issue 2: Does a Started Agent Pod Mean the Pipeline Can Run?

No.  
The Pod is just the carrier; the key is whether the image contains:

- git
- docker
- kubectl
- helm
- and related credentials and network access capabilities

### Issue 3: Can Docker Plugin and Kubernetes Plugin Only Be Used One at a Time?

No.  
They solve different levels of problems, and both often appear in actual environments.

---

## Key Points to Master in This Section

1. Basic architecture of Jenkins on K8s
2. Responsibilities of Controller and Agent
3. Core significance of Agent Pod
4. Boundary between Docker-related capabilities and Kubernetes plugins
5. Executor image content determines execution capabilities

## One-Sentence Summary

When Jenkins runs on Kubernetes, what's truly important isn't just Jenkins itself, but how to dynamically create Agent Pods with the required tools and permissions for build, push, and deploy operations.