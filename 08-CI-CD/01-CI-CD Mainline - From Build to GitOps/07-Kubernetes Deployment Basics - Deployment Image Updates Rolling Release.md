# 06-Jenkins Pipeline to K8s Deployment: From Traditional CI/CD to Containerized Delivery

## Document Notes

This is the 6th note in the 08-CI-CD learning path.

The previous article has already integrated GitLab CI into the current learningMain, with the focus not on memorizing syntax but establishing a understanding:

- Manual build, tag, push, deploy actions you've done
- All these actions can be automated by pipelines

This article continues along the same learning approach, entering Jenkins.

This article does not require you to have a complete Jenkins environment set up now, nor does it require you to master all Jenkinsfile syntax.  
The goals of this article are:

1. Map the manual deployment actions you've done to Jenkins Pipeline
2. Understand the commonalities between Jenkins Pipeline and GitLab CI in the current learningMain
3. Understand what Jenkinsfile does
4. Understand why many enterprises transition from Jenkins to containerized delivery, then further to GitOps

This article continues to align with the current experimental environment and learningMain:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Jenkins #Pipeline #Jenkinsfile #Harbor #kubectl #ContainerizedDelivery #AutomationOfReleases #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understand Jenkins' position in the full deployment chain
2. Understand what a Pipeline is
3. Understand the role of Jenkinsfile
4. Understand how Jenkins handles build, push, deploy actions
5. Understand the commonalities and differences between Jenkins and GitLab CI in the current learningMain
6. Be able to map your previous manual commands to Jenkins Pipeline
7. Be able to read a minimal Jenkinsfile skeleton

## First, Place Jenkins Back Into Your Existing Learning Chain

Previously, you've walked through this chain manually:

Application Content  
→ Image Build  
→ Image Storage  
→ Cluster Configuration  
→ Cluster Deployment  
→ Deployment Verification

The previous article mapped this chain to GitLab CI.

Now ask the same question again:

**What steps does Jenkins automate?**

The answer is very similar to GitLab CI, typically including:

1. Pull code
2. Build application
3. Build image
4. Push to Harbor
5. Update Deployment or execute Helm deployment
6. Check rollout status
7. Perform basic verification or notifications

At this stage, you can first understand Jenkins as:

**Another automation execution framework, which handles the same set of actions as you've done manually before.**

## Part 1: First Understand What Jenkins Actually Does

Jenkins is not a code repository, not Harbor, and not Kubernetes.

Its role is more like an:

**Automation Orchestrator**

That is, its value doesn't lie in inventing new deployment actions, but in connecting many actions together, such as:

- Pull code from Git
- Execute `mvn package`
- Execute `docker build`
- Execute `docker push`
- Execute `kubectl set image`
- Execute `kubectl rollout status`

### Understanding to Establish at This Stage

If you review the actions you've already done, you'll find that Jenkins' work essentially is:

**Executing these commands in sequence and recording the execution results.**

## Part 2: What Is a Pipeline

A Pipeline can be directly understood as:

**An automated workflow.**

In Jenkins, it represents:

- How many steps this deployment has
- What each step does
- The order of execution
- Where to stop on failure
- What to do after success

### How You Should Currently Understand a Pipeline

Don't first think of a Pipeline as a pile of syntax.  
First, think of it as:

**Writing your previously manual deployment steps as a machine-reproducible process.**

For example, you previously did manually:

1. `docker build`
2. `docker tag`
3. `docker push`
4. `kubectl set image`
5. `kubectl rollout status`

A Jenkins Pipeline essentially connects these 5 actions.

## Part 3: What Is a Jenkinsfile

If a Pipeline is "the pipeline itself," then a Jenkinsfile is:

**The definition file for this pipeline.**

Its status is similar to the `.gitlab-ci.yml` from the previous article.

You can think of a Jenkinsfile as:

- A Jenkins pipeline manual
- A file defining stages and specific commands
- A file that solidifies "manual deployment steps" into "automated deployment processes"

### Understanding to Establish at This Stage

A Jenkinsfile is not business code, but it's also not an optional accessory file.  
It answers:

> How exactly does this deployment process run.

## Part 4: Map Your Manual Actions to Jenkins Pipeline

This section is very similar to the previous GitLab CI article, but this is the most important learning approach at this stage.

### Manual Actions You've Already Done

1. `docker build`
2. `docker tag`
3. `docker push`
4. `kubectl set image`
5. `kubectl rollout status`
6. Cluster internal `wget` verification page content

### After Putting These Actions Into Jenkins Pipeline, They Can Usually Be Divided Into Several Stages

- Build
- Push
- Deploy
- Verify

You can also understand it as:

- Build: Build image
- Push: Push to Harbor
- Deploy: Send to K8s
- Verify: Check the update results

### Understanding to Establish at This Stage

A Jenkins Pipeline is not an abstract platform; it's simply receiving the actions you've already done manually.

## Part 5: First Look at a Minimal Jenkinsfile Skeleton

Don't rush to look at complex syntax first; just look at the structure.

    pipeline {
        agent any

        stages {
            stage('Build') {
                steps {
                    echo 'build image'
                }
            }

            stage('Push') {
                steps {
                    echo 'push image to harbor'
                }
            }

stage('Deploy') {
                steps {
                    echo 'deploy to kubernetes'
                }
            }
        }
    }

### What Does This Jenkinsfile Represent

1. This is a Pipeline
2. There is one execution environment: `agent any`
3. There are three stages:
   - Build
   - Push
   - Deploy
4. Each stage contains actions to execute

### Current Understanding Focus

Don't get stuck on syntax details first, focus on the skeleton:

- Jenkins also organizes workflows by stages
- Each stage contains specific commands
- Essentially, it's organizing actions you've already performed manually

## Part 6: Putting Real Commands into Jenkinsfile

Now start putting the actions you've already executed manually into Jenkinsfile for understanding.

### 1) Build Stage

You've already done:

    docker build -t manual-web:v3 .

If placed in Jenkinsfile, it can be understood as:

    stage('Build') {
        steps {
            sh 'docker build -t manual-web:v3 .'
        }
    }

### 2) Push Stage

You've already done:

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301
    docker push harbor.example.com/test/manual-web:dev-c1d2e3f-301

In Jenkinsfile, this becomes:

    stage('Push') {
        steps {
            sh 'docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301'
            sh 'docker push harbor.example.com/test/manual-web:dev-c1d2e3f-301'
        }
    }

### 3) Deploy Stage

You've already done:

    kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:dev-c1d2e3f-301
    kubectl -n test rollout status deployment/manual-web

In Jenkinsfile, this becomes:

    stage('Deploy') {
        steps {
            sh 'kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:dev-c1d2e3f-301'
            sh 'kubectl -n test rollout status deployment/manual-web'
        }
    }

### Current Understanding Focus

Like GitLab CI, Jenkins Pipeline essentially:

**Places the commands you've already executed manually into an automated execution framework.**

## Part 7: Why Jenkins and GitLab CI Feel Repetitive to Learn

Because they both follow the same main line at this learning stage.

### Commonalities

- Both are automation pipeline tools
- Both execute manual commands automatically
- Both have stage concepts
- Both have variables/parameters
- Both integrate with Harbor, K8s, Deployment

### Key Differences

#### GitLab CI

Closely tied to Git repositories, configuration files are usually:

- `.gitlab-ci.yml`

#### Jenkins

More like an independent automation platform, configuration files are usually:

- `Jenkinsfile`

### Current Understanding Focus

For you now, the key isn't to compare which is more advanced, but to understand:

- Both automate the same release chain
- Understanding one's core concepts can be transferred to the other

## Part 8: Why Many Enterprises Start with Jenkins and Gradually Move to Containerized Delivery

This must be explained in the context of real-world environments, otherwise Jenkins would seem abrupt.

Many enterprises didn't start with cloud-native from the beginning, but evolved like this:

### Early Stage

- Manual server login
- Manual package deployment
- Shell script deployment
- Manual service restart

### Intermediate Stage

- Use Jenkins to automate these commands
- Automate build and deployment first
- Still possibly traditional server deployment

### Containerization Stage

- Jenkins starts integrating `docker build`
- Connects to Harbor
- Connects to Kubernetes
- Becomes containerized delivery

### Further Stages

- Later may introduce Helm
- Later may introduce GitOps / Argo CD

### Current Understanding Focus

Jenkins is common not because it's necessarily the most advanced, but because many enterprises' release systems evolved from "traditional automation" step by step.

So learning Jenkins is actually understanding:

**How traditional release systems evolved into containerized delivery.**

## Part 9: How to Best Understand Jenkins in Your Current Context

Currently, the best way for you to understand isn't "first learn all Jenkins plugins."

Instead:

**Jenkins = Using another form to connect the actions I've already performed manually.**

You already know:

- Build images
- Push to Harbor
- Update Deployment
- Rollout status
- Verify

So Jenkins learning focus becomes:

1. How to organize these commands by stages
2. How to write these commands into Jenkinsfile
3. Which values should be extracted into variables/parameters
4. Which credentials shouldn't be hard-coded

This makes the process smooth.

## Part 10: Simulating a Jenkins Pipeline Manually First

This part doesn't require you to actually set up Jenkins, but to practice the "stage feeling" of Jenkins Pipeline.

### Practice Goal

Execute the commands you already know in the following 4 stages in order:

#### Stage 1: Build

Execute:

    docker build -t manual-web:v6 .

docker tag manual-web:v6 your Harbor domain name/test/manual-web:dev-jenkins001-601
docker push your Harbor domain name/test/manual-web:dev-jenkins001-601

#### Stage 3: Deploy

Run:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain name/test/manual-web:dev-jenkins001-601

#### Stage 4: Verify

Run:

    kubectl -n test rollout status deployment/manual-web
    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, run:

    wget -qO- http://manual-web.test.svc.cluster.local

### Current Understanding to Establish

Although you're still executing commands manually, you should start forming a Pipeline perspective in your mind:

- Which stage is this step currently in
- Which stage it should be placed in when writing a Jenkinsfile
- Which values should be made into variables

This step is critical.

## Part 11: Translating the Manual Stages into Jenkinsfile Thinking

The 4 steps you just executed manually can be translated into the following Jenkinsfile structure:

    pipeline {
        agent any

        stages {
            stage('Build') {
                steps {
                    sh 'docker build -t manual-web:v6 .'
                }
            }

            stage('Push') {
                steps {
                    sh 'docker tag manual-web:v6 harbor.example.com/test/manual-web:dev-jenkins001-601'
                    sh 'docker push harbor.example.com/test/manual-web:dev-jenkins001-601'
                }
            }

            stage('Deploy') {
                steps {
                    sh 'kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:dev-jenkins001-601'
                }
            }

            stage('Verify') {
                steps {
                    sh 'kubectl -n test rollout status deployment/manual-web'
                }
            }
        }
    }

### Current Understanding to Establish

At this point you should be able to naturally explain:

**A Jenkinsfile isn't inventing new logic, but rather structuring manual actions.**

## Part 12: Understanding Parameters and Variables in Jenkins

You've already seen variables in GitLab CI.  
Jenkins works the same way, just with different presentation.

At this stage, don't dive deep into syntax - just understand its purpose:

### Purpose 1: Reduce Repetition

For example:

- Harbor address
- Image name
- Tag
- Namespace
- Deployment name

Writing these as fixed values makes future modifications cumbersome.

### Purpose 2: Support Different Environments

For example:

- test
- prod

You'll increasingly need to switch:

- Image tag
- Namespace
- Harbor project
- Replica count
- Domain name

So the purpose of variables or parameters is essentially:

**To make the same pipeline reusable across different contexts.**

## Part 13: What's Most Likely to Go Wrong in Jenkins

This section will be viewed through the lens of your already familiar manual workflow.

### Problem 1: Build Failure

It's essentially the same as manual build failure:

- Dockerfile is wrong
- Content isn't ready
- Build context is incorrect

### Problem 2: Push Failure

It's essentially the same as manual push failure:

- Harbor login failed
- Address is wrong
- Repository permissions are insufficient

### Problem 3: Deploy Failure

It's essentially the same as manual kubectl execution failure:

- kubeconfig is unavailable
- Namespace is wrong
- Image address is wrong
- Harbor image doesn't exist

### Problem 4: Verify Failure

It's essentially the same as previous rollout failure or no change in the page:

- Pod can't pull the image
- Pod isn't Ready
- Service access failed

### Current Understanding to Establish

Jenkins is just an automation executor - it doesn't eliminate these issues.  
It just transforms the problems from "errors during manual execution" to "errors during pipeline execution."

So the troubleshooting experience you had manually remains completely useful.

## Part 14: This Section's Practice Exercise

### Exercise 1: Write a Jenkinsfile Skeleton for the 4 Steps You Just Did

Requirements include:

- `pipeline`
- `stages`
- `Build`
- `Push`
- `Deploy`
- `Verify`

Don't focus on perfect syntax yet - just focus on writing the structure.

### Exercise 2: Answer the Following 4 Questions Yourself

1. What role does Jenkins play in the entire release chain?
2. What is a Pipeline?
3. What is a Jenkinsfile?
4. Why isn't Jenkins inventing new actions, but rather organizing existing ones?

### Exercise 3: Classify All the Commands You Previously Executed into the 4 Stages

Requirements include at least:

- Which command belongs to Build
- Which belongs to Push
- Which belongs to Deploy
- Which belongs to Verify

If you can complete these 3 exercises, you've mastered this section.

## Content You Should Be Able to Explain After This Section

After completing this section, you should be able to clearly explain the following statement:

Jenkins is a category of automation orchestration tool, its core function is to organize previously manually executed build, push, deploy, verify actions into repeatable pipelines.  
Pipeline represents the automation process itself, while Jenkinsfile is the definition file for this process.  
In the current learning path, the actions Jenkins takes and GitLab CI are essentially the same, both focusing on image building, Harbor push, Kubernetes deployment and verification.  
Therefore, learning Jenkins is not about learning a completely new release logic, but about learning how to structure and automate already manually performed actions for repeatable execution.

## Common Issues and Troubleshooting Directions

### Issue 1: Why does Jenkins feel similar to GitLab CI

Because in the current learning phase, they are automating the same release chain.  
Differences lie more in tool ecosystems and organizational approaches, rather than in the release actions themselves.

### Issue 2: Can I still learn without a Jenkins environment

Yes.  
Currently, the most important thing is to first establish the Pipeline mindset:

- How to split stages
- How to categorize commands
- Why variables exist

When you actually get access to an environment, it will become much smoother.

### Issue 3: Will Jenkins be replaced, so it's not worth learning

It's not appropriate to view it this way.  
Many enterprises still extensively use Jenkins, especially in environments transitioning from traditional operations to containerized delivery.  
Learning Jenkins is essentially understanding a common enterprise delivery path.

## Key Takeaways for This Section

After completing this section, you should master:

1. Jenkins' position in the release chain
2. The role of Pipeline and Jenkinsfile
3. How Jenkins handles build, push, deploy, verify actions
4. Similarities and differences between Jenkins and GitLab CI
5. How to map manual commands into Jenkins Pipeline stages

## One-Sentence Summary

The essence of Jenkins Pipeline is to organize already manually completed build, image push, Kubernetes deployment and verification results into a repeatable automated release process.

## Next Section

Next section will enter:

07-K8s Release Basics: Deployment, Image Updates and Rolling Release Mechanisms

The next section will refocus the perspective inside Kubernetes clusters, emphasizing understanding:

- Why changing an image triggers a rolling update
- The relationship between Deployment, ReplicaSet, and Pod
- What rollout status/history/undo are viewing
- Why release success cannot be judged solely by Pod Running /think