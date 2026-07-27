# 14-GitLab Runner and Jenkins Agent Basics: How Pipeline Executors Work

## Document Description

This article is the 14th note in the 08-CI-CD learning pathway.

In previous sections 01-13, we established a minimal release pipeline and gradually covered the following components:

- Image building
- Image tagging
- Harbor repository
- GitLab CI
- Jenkins Pipeline
- K8s rolling updates
- Helm
- Multi-environment deployment

In this article, we will introduce a very crucial but often overlooked component:

**Pipeline executors.**

Many people learning GitLab CI or Jenkins tend to focus on the “definition layers” such as `.gitlab-ci.yml`, Jenkinsfile, stages, jobs, and pipelines.

However, during actual execution, it’s not the GitLab or Jenkins interfaces that run these commands. Instead, it’s:

- GitLab Runner
- Jenkins Agent

The focus of this article is to explain these execution layers in detail and show how they correspond to the manual commands you have previously executed.

This article continues to use the current experimental setup and environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- The minimum application `manual-web` has already been built, pushed, deployed, and verified.

## Tags

#Kubernetes #CI-CD #GitLabRunner #JenkinsAgent #Pipeline Executors #Harbor #Docker #kubectl #PracticalNotes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what GitLab Runner and Jenkins Agent are.
2. Comprehend why pipelines require executors.
3. Distinguish between “pipeline definition” and “pipeline execution” as two separate concepts.
4. Identify the tools typically needed on Runners / Agents.
5. Use the commands you have executed previously to understand the role of executors in actual deployment processes.
6. Explain why without executors, even a complete CI/CD configuration will not function.

## Experimental Focus for This Article

At this stage, you don’t need to install GitLab Runner or Jenkins Agent and integrate them into your platform. The focus is on three key tasks:

1. Separate the “definition layer” from the “execution layer.”
2. Use the commands you have manually executed to understand what executors do.
3. Conduct a “minimal local script execution experiment” to simulate the working behavior of Runners / Agents.

---

## Part 1: Answering a Core Question – Who Really Executes the Commands in the Pipeline?

You have already seen that in GitLab CI, there are:

- `.gitlab-ci.yml`
- Stages
- Jobs
- Scripts

And in Jenkins, there are:

- Jenkinsfile
- Pipelines
- Stages
- Steps
- `sh '...'`

These components all define what needs to be done, in what order, but they don’t actually execute the commands themselves.

For example, if you write this in GitLab CI:

    script:
      - docker build -t manual-web:v10 .

or in Jenkinsfile:

    sh 'docker build -t manual-web:v10 .'

It’s not the web interface that executes these commands. Instead, it’s a real execution environment, namely:

- GitLab Runner
- Jenkins Agent

### Understanding to Establish at This Step

Pipelines consist of two layers:

#### Layer 1: Definition Layer

Examples include:

- `.gitlab-ci.yml`
- Jenkinsfile

#### Layer 2: Execution Layer

Examples include:

- GitLab Runner
- Jenkins Agent

Any understanding of CI/CD must distinguish between these two layers.

---

## Part 2: What is GitLab Runner?

You can think of GitLab Runner as a “worker node” dedicated to executing jobs in GitLab CI. Its responsibilities include:

1. Receiving jobs assigned by GitLab.
2. Preparing the execution environment.
3. Pulling the code.
4. Executing the commands specified in `.gitlab-ci.yml` scripts.
5. Returning logs and results to GitLab.

### A More Straightforward Explanation

GitLab Runner is like a “worker” that takes on tasks assigned by GitLab. GitLab itself acts more like a “scheduling center” that:

- Monitors code changes.
- Reads `.gitlab-ci.yml` files.
- Generates pipelines.
- Decides which jobs to execute.

Runner is the one that actually runs the commands, builds the applications, pushes them, and deploys them.

### Understanding to Establish at This Step

For GitLab CI to function properly, it depends not only on whether `.gitlab-ci.yml` is correctly written but also on whether there is a Runner available, whether the Runner can handle the tasks, and whether it has the necessary tools.

---

## Part 3: What is Jenkins Agent?

Jenkins Agent can### Summary

This chapter has provided an in-depth understanding of CI/CD execution layers, focusing on Runner and Agent. We have learned that these components are essential for executing pipeline commands and that their capabilities directly affect the success of CI/CD processes. The chapter also highlighted the importance of having the right tools and permissions in the execution environment.

In summary, GitLab Runner and Jenkins Agent serve as the executors that carry out the defined pipeline steps. Successful execution depends on whether the executor has the necessary tools, such as Docker, Harbor credentials, kubectl, and access to K8s clusters. Understanding these differences and ensuring that the executor meets the minimum requirements is crucial for a smooth CI/CD flow.