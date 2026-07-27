# 05-GitLab CI Basics: .gitlab-ci.yml, Stage, Job, and Variables

## Document Description

This article is the fifth note in the 08-CI-CD learning pathway.

In previous articles, we have broken down the manual deployment process step by step:

1. Manually building and deploying a minimal application to Kubernetes.
2. Dividing the deployment process into several fixed stages.
3. Understanding image building and image tag design.
4. Understanding projects, repositories, tags, and Robot accounts in Harbor.

In this article, we will introduce the first form of automation:

**GitLab CI.**

This article does not require you to create a complete pipeline from scratch or connect all environments at once.  
The objectives are:

- Map the actions you have already performed manually to GitLab CI.
- Understand what `.gitlab-ci.yml` controls.
- Recognize the roles of Stage, Job, and variables.
- Use a minimal example to demonstrate how automation simplifies certain steps.

This article continues following the current learning focus. The default environment is as follows:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace for experimentation

## Tags

#Kubernetes #CI-CD #GitLabCI #gitlab-ci.yml #Stage #Job #Variables #Harbor #Automated Deployment #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand GitLab CI’s role in the entire deployment process.
2. Know what `.gitlab-ci.yml` does.
3. Distinguish between Stage and Job.
4. Recognize the importance of variables.
5. Map the manual actions you performed previously to GitLab CI.
6. Write a basic GitLab CI configuration.
7. Explain which steps of the deployment process will be automated by GitLab CI.

## Reconnecting GitLab CI to the Existing Process

You have already completed this process manually:

Application content → Image building → Image storage → Cluster configuration → Cluster deployment → Deployment verification

Before learning new syntax, let’s ask a question:

**Which steps of this process will GitLab CI automate?**

At this stage, the most common automated tasks are in the first half of the process:

1. Pulling code.
2. Building the application.
3. Building the image.
4. Pushing the image to Harbor.

Later on, additional automation can be applied:

5. Updating configurations.
6. Deploying to Kubernetes.
7. Performing basic verification.

So, you can think of GitLab CI as:

**An automated framework that executes the fixed commands you previously ran manually.**

## Part 1: Understanding What GitLab CI Does

### Core Understanding

GitLab CI itself is not code, nor Harbor, nor Kubernetes.  
Its role is more like an automation execution hub:

- It detects changes in the code.
- Executes commands according to your predefined rules.
- Displays logs and results.
- Determines whether the automation process was successful or failed.

### Key Concepts to Understand

You have previously performed these manual tasks:

- `docker build`
- `docker tag`
- `docker push`
- `kubectl set image`
- `kubectl rollout status`

GitLab CI essentially does the same thing:

**It automatically runs these manual tasks in sequence.**

## Part 2: What `.gitlab-ci.yml` Is

`.gitlab-ci.yml` is the core configuration file for GitLab CI.

You can think of it as:

**The instruction manual for this automation pipeline.**

It is usually placed in the root directory of the code repository and tells GitLab:

- How many stages the pipeline has.
- What tasks are included in each stage.
- Which commands will be executed for each task.
- Which variables need to be used.
- When each task should be run.

### Key Concepts to Understand

Without `.gitlab-ci.yml`, GitLab would not know what automated actions you want to perform.  
Therefore, it is not a trivial configuration but:

**The entry point for the automated deployment process.**

## Part 3: What Stage and Job Are

These are two fundamental concepts in GitLab CI.

### 1) Stage

A Stage can be understood as:

**A major segment of the pipeline.**

For example:

- build
- image
- deploy

Their role is to determine:

- Which tasks should be done first.
- Which tasks follow after that.
- How the process is structured in layers.

### 2) Job

A Job can be understood as:

**A specific task within a Stage.**

For example:

- `build_app`
- `build_image`
- `push_harbor`
- `deploy_test`

Their role is to specify:

- What exact commands need to be executed.
- What the name of this task is.
- To which Stage it belongs.

### Key Concepts to Understand

You can remember it this way        - docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${HARBOR_REGISTRY}/${HARBORPROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
        - docker push ${HARBOR_REGISTRY}/${HARBOR PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}

### Understanding Required for This Step

Variables have been used to consolidate the following "frequently recurring values":

- Harbor address
- Harbor project
- Image name
- Tag

Once you change these variables, you don't need to modify the actual commands everywhere.

## Part 9: First, Conduct a "Manual Simulation of GitLab CI" Exercise Based on Your Current Learning Environment

You may not have a fully operational GitLab CI environment yet, but that's okay.  
For now, focus on something more important:

**Understand each Job in GitLab CI as "scripted manual actions."**

### Exercise Objectives

Manually perform the following three sets of actions and identify which Jobs they would correspond to in GitLab CI.

#### Action 1: Build the Image

Command to execute:

    docker build -t manual-web:v5 .

Understanding required:

- This will eventually belong to a Job like `build_image`.

#### Action 2: Push to Harbor

Command to execute:

    docker tag manual-web:v5 your-Harbor-domain/test/manual-web:dev-test001-501
    docker push your-Harbor-domain/test/manual-web:dev-test001-501

Understanding required:

- This will eventually belong to a Job like `push_image`.

#### Action 3: Update the Deployment

Command to execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-test001-501

Understanding required:

- This will eventually belong to a Job like `deploy_test`.

### Understanding Required for This Step

Here, you're not repeating previous experiments; instead, you're establishing a mapping relationship:

- Which Job each manual command will be part of.
- Which Stage each Job should be placed in.
- Which values should be defined as variables.

After completing this step, GitLab CI will become much clearer to you when you study it further.

## Part 10: Write a Minimal GitLab CI Example That Aligns More Closely with Your Current Learning Focus

You don't need to run this configuration immediately; just understand its structure first.

    stages:
      - image
      - deploy

    variables:
      HARBOR_REGISTRY: "harbor.example.com"
      HARBORPROJECT: "test"
      IMAGE_NAME: "manual-web"
      IMAGE_TAG: "dev-c1d2e3f-301"
      NAMESPACE: "test"
      DEPLOY_NAME: "manual-web"

    build_and_push_image:
      stage: image
      script:
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker tag ${IMAGE_NAME}:${IMAGE_tag} ${HARBOR_REGISTRY}/${HARBORPROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
        - docker push ${HARBORRegistry}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}

    deploy_test:
      stage: deploy
      script:
        - kubectl -n ${NAMESPACE} set image deployment/${DEPLOY_NAME} ${IMAGE_NAME}=${HARBOR_REGISTRY}/${HARBORPROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
        - kubectl -n ${NAMESPACE} rollout status deployment/${DEPLOY_NAME}

### How to Interpret This Configuration

Don't rush to memorize the syntax details; focus on the structure first:

#### Stages

- `image`
- `deploy`

#### Jobs

- `build_and_push_image`
- `deploy_test`

#### Variables

- Harbor address
- Project name
- Image name
- Tag
- Namespace
- Deployment name

#### Script

The script contains all the commands you have manually executed before.

### Understanding Required for This Step

By now, you should be able to articulate:

**GitLab CI doesn't do anything new and mysterious; it simply puts the build, push, and deploy commands I previously performed manually into an automated execution framework.**

## Part 11: Why GitLab CI Must Have a Runner

We won't discuss the implementation details in this section; instead, focus on the understanding you need to develop at this stage.

The GitLab CI interface itself doesn't execute `docker build`.  
It is the Runner that actually runs these commands.

You can think of a Runner as:

**The execution machine for the pipeline.**

Without a Runner, here would be a problem:

- GitLab sees the `.gitlab-ci.yml` file and knows which Jobs need to be run.
- But there is no actual executor, so the Jobs cannot be executed.

### Understanding Required for This Step

As you continue learning about GitLab CI, remember:

- `.gitlab-ci.yml` defines the process.
- The Runner is responsible for actually executing the commands.

##- What do "Pipeline" and "Jenkinsfile" correspond to in the current main line respectively?
- Why do many enterprises gradually evolve from using Jenkins to a more complete cloud-native delivery system?