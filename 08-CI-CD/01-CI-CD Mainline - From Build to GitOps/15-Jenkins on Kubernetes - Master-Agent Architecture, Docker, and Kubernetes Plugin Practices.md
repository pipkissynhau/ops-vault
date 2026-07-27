# 15-Jenkins on Kubernetes: Master-Agent Architecture, Docker, and Kubernetes Plugin Practices

## Document Description

This article is the 15th note in the 08-CI-CD learning pathway.

The previous article clarified a key point:

- Both GitLab Runner and Jenkins Agent are essentially pipeline executors.
- Pipeline configuration serves only as a “definition layer.”
- The actual execution of build, push, and deploy tasks occurs in the environment where the executor runs.

This article builds upon that by addressing the following question:

**If Jenkins itself is running within Kubernetes, how does it perform pipeline execution?**

This article doesn’t expect you to set up a fully production-grade Jenkins on K8s system today. Instead, the focus is on establishing the core structure and the minimum practical steps, so you understand:

- What Jenkins typically looks like in K8s.
- The relationship between Master/Controller and Agent.
- Why Docker and Kubernetes plugins exist.
- Why Jenkins Agents in K8s are often temporary Pods.
- How this model aligns with the Runner/Agent concepts you learned earlier.

This article continues to follow the current learning framework:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Completed minimal application build/push/deploy/rollback/Helm deployment experiments

## Tags

#Kubernetes #CI-CD #Jenkins #JenkinsAgent #MasterAgent #KubernetesPlugin #DockerPlugin #PipelineExecutor #PracticalNotes

## Learning Objectives

After completing this article, you should be able to:

1. Understand Jenkins’s basic architecture in K8s.
2. Grasp the relationship between Master (or Controller) and Agent.
3. Explain why Jenkins Agents often run as Pods in K8s.
4. Distinguish the roles of Docker and Kubernetes plugins.
5. Apply the concept of “executor” learned earlier to the Jenkins on K8s context.
6. Justify why many enterprises choose to run Jenkins within K8s.

## Experimental Approach for This Article

This article is divided into 4 sections:

1. Establish an initial understanding of Jenkins’s structure in K8s.
2. Use minimal YAML and Pipeline concepts to comprehend Agent Pods.
3. Understand the responsibilities of Docker and Kubernetes plugins.
4. Conduct a small experiment to simulate the role of an Agent from a local perspective.

---

## Section 1: Understanding Jenkins’s Basic Architecture in K8s

Let’s start by building a basic understanding of its structure.

When Jenkins runs in K8s, it typically involves at least two types of components:

### 1) Jenkins Controller (formerly known as Master)

Its responsibilities include:

- Providing the Jenkins web interface.
- Managing Jobs and Pipelines.
- Managing credentials.
- Scheduling tasks.
- Deciding which Agent will execute tasks.
- Recording logs and build history.

### 2) Jenkins Agent

Its responsibilities are:

- Actually executing the commands within the pipeline.
- Pulling code.
- Performing builds.
- Pushing results to Harbor.
- Running `kubectl` or Helm commands.
- Transmitting logs and outcomes.

### Key Points to Understand at This Stage

You can think of it this way:

- The Controller manages everything.
- The Agent performs the actual tasks.

This aligns perfectly with the executor concept you learned earlier.

---

## Section 2: Why Doesn’t Jenkins Execute All Commands Itself?

This is a crucial question.

Theoretically, the Jenkins Controller could handle everything:

- Providing the interface.
- Scheduling tasks.
- Performing builds.
- Pushing results.
- Deploying applications.

However, this would quickly lead to problems:

### Problem 1: Mixed Responsibilities

If the Controller manages both system operations and builds, it will become increasingly heavy when there are many build tasks.

### Problem 2: Lack of Flexibility in the Environment

Different pipelines may require different execution environments, such as Maven, Node.js, Docker, `kubectl`, or Helm. If all these are managed by the Controller, the environment will become unwieldy.

### Problem 3: Poor Isolation

If one build task fails, it could affect the entire Jenkins system.

### Key Points to Understand at This Stage

The correct approach is usually:

**The Controller manages, and the Agent executes.**

---

## Section 3: Why Are Temporary Agent Pods Common in Jenkins on K8s?

This section illustrates how Jenkins integrates with Kubernetes.

In traditional Jenkins setups, Agents might be:

- A fixed physical machine.
- A fixed virtual machine.
- A dedicated Docker host.

However, in K8s, it’s more natural to use temporary Agent Pods:

### Benefits of Using Temporary Pod Agents

#### 1) Temporary Nature

After a pipeline completes, the Agent Pod can be terminated. This eliminates the need to maintain many idle execution nodes.

#### 2) Greater Flexibility in the Environment

Different pipelines can use different Agent### Summary

This chapter explores how Jenkins on Kubernetes dynamically manages its execution agents through Pods, allowing for flexible resource allocation and tool customization. The core concept is to separate the management layer (Jenkins Controller) from the execution layer (Agent Pod(s)), with different plugins addressing specific tasks such as image building, deployment, and security management. By understanding these components and their interactions, users can better leverage Jenkins on Kubernetes for efficient and scalable pipeline operations.