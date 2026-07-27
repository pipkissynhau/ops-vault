# 06-Jenkins Pipeline to K8s Release: From Traditional Pipelines to Containerized Delivery

## Document Description

This article is the sixth note in the 08-CI-CD learning pathway.

The previous article introduced GitLab CI into our current learning framework. The focus was not on memorizing the syntax but on gaining an understanding of:

- The build, tag, push, and deploy actions that you have performed manually
- How these actions can be automated by pipelines

This article continues along the same learning approach by introducing Jenkins.

There is no requirement for you to set up a complete Jenkins environment or master all Jenkinsfile syntax right away.  
The goals of this article are:

1. To map the release actions you have performed manually into Jenkins Pipelines
2. To understand the similarities between Jenkins Pipelines and GitLab CI in our current learning context
3. To comprehend what Jenkinsfile does
4. To grasp why many enterprises transition from Jenkins to containerized delivery and further to GitOps

This article continues to align with our current experimental environment and learning framework:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Jenkins #Pipeline #Jenkinsfile #Harbor #kubectl #Containerized Delivery #Automated Release #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand Jenkins's role in the entire release process
2. Define what a Pipeline is
3. Recognize the purpose of Jenkinsfile
4. Understand how Jenkins handles build, push, and deploy actions
5. Distinguish the similarities and differences between Jenkins and GitLab CI in our current learning context
6. Map the manual commands you previously used into Jenkins Pipelines
7. Interpret a basic Jenkinsfile structure

## First: Reintroduce Jenkins into the Process You Have Learned

Previously, you have experienced this process manually:

Application content  
→ Image building  
→ Image storage  
→ Cluster configuration  
→ Cluster deployment  
→ Release verification

The previous article also mapped this process to GitLab CI.

Let's ask the same question again:

**Which steps will Jenkins automate?**

The answers are similar to those for GitLab CI, typically including:

1. Pulling code from Git
2. Building the application
3. Creating an image
4. Pushing the image to Harbor
5. Updating the Deployment or executing Helm for deployment
6. Checking the rollout status
7. Performing basic verification or notifications

So, at this stage, you can think of Jenkins as:

**Another automation framework that performs the same steps you have done manually.**

## Part 1: Understanding What Jenkins Does

Jenkins is not a code repository, nor is it Harbor or Kubernetes.

Its role is more like that of an:

**Automation orchestrator**

In other words, its value lies in connecting various actions together, such as:

- Pulling code from Git
- Running `mvn package`
- Executing `docker build`
- Performing `docker push`
- Running `kubectl set image`
- Checking `kubectl rollout status`

### Understanding to Establish at This Step

If you look at the actions you have performed manually again, you will see that Jenkins essentially does the following:

**It executes these commands in sequence and records the results.**

## Part 2: What is a Pipeline?

A Pipeline can be directly understood as:

**An automated workflow.**

In Jenkins, it represents:

- How many steps are involved in this release process
- What each step entails
- Which actions should be executed first and which later
- Where to stop in case of failure
- What to do after successful completion

### How You Should Understand a Pipeline at This Point

Don’t focus on the syntax for now.  
Think of a Pipeline as:

**A written sequence of steps that you have performed manually, converted into a machine-readable process.**

For example, if you have previously done the following manually:

1. `docker build`
2. `docker tag`
3. `docker push`
4. `kubectl set image`
5. `kubectl rollout status`

Then a Jenkins Pipeline would essentially connect these five steps together.

## Part 3: What is Jenkinsfile?

If a Pipeline represents “the workflow itself,” then Jenkinsfile is:

**The definition document for this workflow.**

Its role is similar to that of the `.gitlab-ci.yml` file in the previous article.

You can think of Jenkinsfile as:

- Jenkins’s instruction manual for the pipeline
- A file that specifies each stage and the specific commands to be executed
- A document that transforms “manual release steps” into an “automated delivery process”

### Understanding to Establish at This Step

Jenkinsfile is not business code, but it is also essential.  
It provides answers to the question:

```bash
sh 'kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:dev-c1d2e3f-301'
sh 'kubectl -n test rollout status deployment/manual-web'
```

### Current Understanding to Be Established

Just like GitLab CI, Jenkins Pipeline is essentially:

**putting the commands you have already performed manually into an automated execution framework.**

## Section 7: Why Jenkins and GitLab CI May Seem Repeated When Learning Them

At this stage of learning, they both follow the same main thread.

### Their Commonalities

- Both are automation pipeline tools.
- Both automatically execute manual commands.
- Both have the concept of stages.
- Both use variables/parameters.
- Both integrate with Harbor, Kubernetes, and Deployments.

### Their Main Differences

#### GitLab CI

It is closer to Git repositories, and its configuration file is usually:

- `.gitlab-ci.yml`

#### Jenkins

It is more like an independent automation platform, and its configuration file is usually:

- `Jenkinsfile`

### Current Understanding to Be Established

For you right now, the focus should not be on which one is more advanced, but rather on understanding that:

- Both automate the same release process.
- The core concepts learned with one can be applied to the other.

## Section 8: Why Many Enterprises Start with Jenkins and Gradually Move Towards Containerized Delivery

This needs to be explained in context; otherwise, Jenkins might seem out of place.

Many enterprises do not start their delivery processes directly with cloud-native technologies. Instead, they evolve in the following way:

### Early Phase

- Manually log in to servers.
- Manually release packages.
- Use Shell scripts for deployment.
- Manually restart services.

### Intermediate Phase

- Automate these commands using Jenkins.
- First, automate building and releasing.
- But still, it may be traditional server-based releases.

### Containerization Phase

- Jenkins begins to integrate with `docker build`.
- Connects with Harbor.
- Integrates with Kubernetes.
- Starts moving towards containerized delivery.

### Further Phases

- Later on, Helm might be introduced.
- Then GitOps or Argo CD might be added.

### Current Understanding to Be Established

Jenkins is widely used not because it is necessarily the most advanced, but because many enterprises' release systems have evolved from "traditional automation" step by step.

Therefore, learning Jenkins is also about understanding:

**how traditional release systems have evolved into containerized delivery.**

## Section 9: How to Understand Jenkins in Your Current Context

The best way to understand Jenkins right now is not to try to learn all its plugins first.

Instead, think of it as:

**Jenkins = another form of organizing the actions you have already been performing manually.**

You already know how to do the following:

- Build images.
- Push them to Harbor.
- Update Deployments.
- Check rollout status.
- Verify results.

So, the focus of learning Jenkins should be on:

1. How these commands are organized into stages.
2. How to write them in a Jenkinsfile.
3. Which values should be defined as variables or parameters.
4. Which credentials should not be hard-coded.

This way, it will make more sense.

## Section 10: Manually Simulate the Idea of a Jenkins Pipeline

In this section, you don't need to actually set up Jenkins; instead, focus on understanding the "stage concept" of a Jenkins Pipeline.

### Practice Objective

Follow the order of these 4 steps and re-execute the commands you already know:

#### Stage 1: Build

Execute:

    docker build -t manual-web:v6 .

#### Stage 2: Push

Execute:

    docker tag manual-web:v6 your_Harbor_domain_name/test/manual-web:dev-jenkins001-601
    docker push your_Harbor_domain_name/test/manual-web:dev-jenkins001-601

#### Stage 3: Deploy

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your_Harbor_domain_name/test/manual-web:dev-jenkins001-601

#### Stage 4: Verify

Execute:

    kubectl -n test rollout status deployment/manual-web
    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the curl-test command, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Current Understanding to Be Established

Although you are still executing commands manually, you should start thinking in terms of a pipeline:

- Which stage this step belongs to.
- Where it should be placed in a Jenkinsfile.
- Which values should be defined as variables.

This step is very important.

## Section 11: Translate the Above Manual Steps into Jenkinsfile Thinking

The 4 steps you just executed can be translated into- Which one belongs to Deploy?
- Which one belongs to Verify?

If you can complete all three exercises, you will have mastered this section.

## Key Points to Discuss After Completing This Section

After completing this section, it is recommended that you be able to clearly explain the following content:

Jenkins is a type of automated deployment and orchestration tool. Its core function is to organize previously manual tasks such as build, push, deploy, and verify into repeatable pipelines.  
A Pipeline represents the automated process itself, while Jenkinsfile is the definition document for this process.  
In the current learning context, the actions that Jenkins handles are essentially the same as those performed by GitLab CI, both focusing on image building, Harbor pushing, Kubernetes deployment, and verification.  
Therefore, learning Jenkins is not about learning a completely new set of deployment logic but rather about how to structure, automate, and make these manual tasks repeatable.

## Common Questions and Troubleshooting Directions

### Question 1: Why do Jenkins and GitLab CI seem so similar?

At this stage of learning, they are both automating the same deployment process.  
The differences lie more in the tool ecosystems and organizational methods rather than the specific deployment actions themselves.

### Question 2: I don’t have a Jenkins environment yet; can I still learn it?

Yes, you can.  
The most important thing now is to develop an understanding of Pipeline concepts:

- How to break down tasks into stages
- How to categorize commands
- Why variables are used

Once you actually set up a Jenkins environment, things will become much clearer.

### Question 3: Will Jenkins be phased out in the future, so there’s no need to learn it?

This is not a valid concern.  
Many companies still use Jenkins extensively, especially those transitioning from traditional operations to containerized delivery.  
Learning Jenkins helps you understand a common enterprise delivery approach.

## Key Takeaways

After completing this section, you should understand:

1. Jenkins’s role in the deployment process
2. The functions of Pipeline and Jenkinsfile
3. How Jenkins integrates build, push, deploy, and verify tasks
4. The similarities and differences between Jenkins and GitLab CI
5. How to map manual commands into Jenkins Pipeline stages

## Summary in One Sentence

The essence of Jenkins Pipeline is to organize previously manual deployment tasks—build, image pushing, Kubernetes deployment, and verification—into a repeatable automated process.

## Next Section

In the next section, we will focus on the basics of Kubernetes deployment: Deployment, image updates, and the rolling update mechanism.  
This section will delve deeper into the Kubernetes cluster, exploring:

- How changing an image triggers a rolling update
- The relationships between Deployment, ReplicaSet, and Pod
- What rollout status, history, and undo indicate
- Why successful deployment cannot be judged solely by whether Pods are running