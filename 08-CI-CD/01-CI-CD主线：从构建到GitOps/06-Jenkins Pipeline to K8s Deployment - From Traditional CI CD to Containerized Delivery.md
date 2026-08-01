# 06-Jenkins Pipeline to K8s Deployment: From Traditional Pipelines to Containerized Delivery

## Documentation Notes

This is the 6th learning note in the 08-CI-CD directory.

The previous articles have gradually established the main line:

1. Why Kubernetes eventually moves toward CI/CD
2. What a complete delivery pipeline looks like
3. How applications transform from source code to images
4. Harbor's position in the image delivery chain

This article enters a very practical topic:

**How Jenkins pushes "code changes" step by step into a Kubernetes cluster.**

Many DevOps, platform, and operations positions, even if their companies have already adopted containerization and cloud-native practices, still heavily use Jenkins. The reason is simple:

- Heavy historical legacy
- Mature ecosystem
- Strong extensibility
- Many teams evolve from traditional deployment to containerized deployment
- Jenkins is often the first established automation deployment entry in enterprises

So even if you later encounter GitLab CI, GitHub Actions, or Argo CD, understanding this Jenkins line remains valuable.

This article's goals are:

1. Understand Jenkins' position in the entire delivery pipeline
2. Understand what a Pipeline is
3. Understand how Jenkins connects building, images, Harbor, and K8s deployment
4. Understand the common ways Jenkins connects to K8s deployment
5. Lay the foundation for the subsequent practical guide on "from image building to deployment to K8s"

## Tags

#Kubernetes #CI-CD #Jenkins #Pipeline #Harbor #Docker #kubectl #Helm #ContainerizedDelivery #Autopublishing

## First answer a fundamental question: What does Jenkins do in the pipeline

Recover the overall pipeline first:

Code → Build → Image → Repository → Configuration → Cluster → Validation → Rollback

Jenkins is not a single-point tool in this process, it's more like:

**An automation orchestrator that connects multiple stages.**

In other words, Jenkins itself typically doesn't directly equal:

- Code repository
- Image repository
- Kubernetes cluster

But it can act as a "central control panel" connecting these systems:

- Pull code from Git
- Execute builds
- Execute docker build
- Login to Harbor and push images
- Execute kubectl apply or helm upgrade
- Call scripts for deployment validation

So you can first remember this sentence:

> Jenkins' core value isn't a specific capability, but rather automating the delivery process.

## Why many companies still heavily use Jenkins

Many beginners have a misconception that Jenkins is an "old tool" that seems outdated.

This view is inaccurate.

In real enterprise environments, Jenkins is still very common, especially in these scenarios:

### 1) Many legacy systems

Many companies didn't start building their delivery systems from the cloud-native era. Instead, they first had:

- VM deployment
- Shell deployment scripts
- Java application deployment
- Artifact management
- SSH deployment

Later they gradually integrated Docker, Harbor, and K8s.

Jenkins is very suitable for taking on the role of "evolving from traditional deployment to containerized deployment."

---

### 2) Strong extensibility

Jenkins has a rich plugin ecosystem that can connect various systems:

- Git
- Maven
- Gradle
- Docker
- Kubernetes
- SonarQube
- Harbor
- Slack / Email / Webhook notifications

So many companies don't want to rebuild their entire delivery platform at once, so they continue to evolve based on Jenkins.

---

### 3) Easier for operations and platform teams to take over

Jenkins often has a strong "script-like" feel in many scenarios, making it easier for operations and platform engineers to integrate:

- Shell
- Docker
- kubectl
- Helm
- SSH
- Credential management

This is actually more friendly for those with an operations background.

## What is a Pipeline

Pipeline can be directly translated as "pipeline."

But don't just understand it as an abstract term; in Jenkins, it has a very clear meaning:

**Pipeline = A complete automation deployment process described by code.**

In other words, previously you might manually do these things:

1. Pull code
2. mvn package
3. docker build
4. docker push
5. kubectl apply
6. Check deployment status

The goal of a Pipeline is to define this entire sequence of actions as a maintainable, repeatable process definition.

## Why use Pipeline instead of just clicking Jenkins pages

Jenkins had many tasks configured manually through the page in its early days, but later increasingly emphasized Pipelines, for very practical reasons.

### 1) Clearer process

You can directly see the complete deployment steps, rather than scattered in page configurations.

---

### 2) Easier version control management

Pipeline configurations can be committed to Git repositories alongside code, such as the common:

    Jenkinsfile

This way, the deployment process itself can be tracked, audited, and rolled back.

---

### 3) More suitable for complex processes

For example, you need to do:

- Multi-stage builds
- Conditional judgments
- Different deployment logic for different branches
- Parallel execution
- Manual approvals
- Notifications after failure

These are more naturally expressed using Pipeline code.

---

### 4) More suitable for team collaboration

Everyone can see what the pipeline actually does, rather than relying on someone to maintain it by clicking through the Jenkins page.

## Typical Position of Jenkins Pipeline in the Delivery Chain

Putting it back into the overall pipeline, Jenkins Pipeline typically covers these stages:

- Code pull
- Build
- Unit testing
- Image building
- Image pushing
- Deployment execution
- Simple validation
- Notifications

In other words, Jenkins may manage the entire process from "code entering the pipeline" all the way to "application entering K8s."

This is why many companies have Jenkins as the central axis of their delivery pipeline.

## What does a typical Jenkins to K8s deployment pipeline look like

First, here's the most common overall path.

### Typical Pipeline

1. Developer commits code to Git
2. Jenkins detects code changes or is triggered by a webhook
3. Jenkins pulls code
4. Jenkins executes Maven / Gradle build
5. Jenkins builds Docker image
6. Jenkins logs in to Harbor and pushes the image
7. Jenkins updates deployment configuration or directly executes deployment commands
8. Kubernetes pulls the new image and performs a rolling update
9. Jenkins checks deployment status
10. Jenkins notifies deployment results

You can remember this pipeline as a single sentence:

> Jenkins turns "manual deployment steps" into "automated execution steps."

## The Most Common Stages of a Jenkins Pipeline

In the learning phase, you can first understand Jenkins Pipeline as consisting of multiple stages.

The most common stages are approximately:

- Checkout
- Build
- Test
- Image Build
- Push
- Deploy
- Verify
- Notify

---
## Stage 1: Checkout

This stage is responsible for pulling code.

Common actions include:

- Pulling code from GitLab / GitHub / Gitee
- Switching to the specified branch
- Retrieving current commit information

The goal of this stage is:

**To ensure the pipeline gets the source code to process this time.**

If the code isn't pulled at all, subsequent builds are meaningless.

---

## Stage 2: Build

This stage is responsible for transforming source code into build artifacts.

For example in Java scenarios:

- `mvn clean package -DskipTests`
- `gradle build`

The output is typically:

- jar packages
- war packages
- or other compiled artifacts

The goal of this stage is:

**To turn code into runnable artifacts first.**

---

## Stage 3: Test

This stage is responsible for testing or quality checks.

For learning purposes, you can initially understand it as:

- Unit tests
- Basic checks
- Static scanning
- Code quality validation

In many enterprises, this stage is often merged with Build, and some projects may simplify it or skip it entirely.

But from an engineering perspective, it holds value.

---

## Stage 4: Image Build

This stage is responsible for turning build artifacts into Docker images.

For example:

- Execute `docker build` based on Dockerfile
- Generate new image tags

This step truly connects:

Code / jar package → Image

---

## Stage 5: Push

This stage is responsible for pushing images to Harbor.

Common actions include:

- `docker login`
- `docker push`
- Using Harbor credentials or Robot account authentication

After this stage, images are no longer just local Jenkins node images, but become:

**A unified delivery artifact that K8s clusters can pull.**

---

## Stage 6: Deploy

This stage is responsible for sending the new version into Kubernetes.

Common implementation methods are three:

### Method 1: Direct kubectl apply

Jenkins executes in the pipeline:

- Modify the image tag in YAML
- `kubectl apply -f`

Suitable for:

- Simple environments
- Learning entry
- Native YAML management scenarios

---

### Method 2: Using helm upgrade

Jenkins executes in the pipeline:

- Update values
- `helm upgrade --install`

Suitable for:

- Applications with many resources
- Significant parameter differences across environments
- Already using Helm for deployment templates

---

### Method 3: Modify GitOps repository, then hand it to Argo CD

Jenkins doesn't directly operate the cluster, but:

- Update the image version in the GitOps repository
- Commit Git
- Argo CD is responsible for synchronizing to the cluster

Suitable for:

- More engineering GitOps mode
- Wanting cluster state to be Git-based

This article will focus on the first two methods, especially Jenkins directly cooperating with kubectl/Helm.

---

## Stage 7: Verify

This stage is responsible for basic release verification.

For example:

- `kubectl rollout status`
- Check Pod Ready
- Simple curl health check
- Basic API check

Remember:

**Pipeline success does not mean business success.**

So even the simplest Pipeline should have basic verification actions.

---

## Stage 8: Notify

This stage is responsible for informing relevant personnel or systems about the release result.

For example:

- Enterprise WeChat notification
- Email notification
- Slack notification
- Webhook notification

This isn't "fluff," but part of the delivery loop.

Because release is a team action, not just the pipeline knowing the result is enough.

## What is a Jenkinsfile

If Pipeline is "release process," then Jenkinsfile is:

**The definition file of this process.**

It is usually placed in the root directory of the code repository.

For example:

    Jenkinsfile

This way, code and pipeline definition can be managed together.

## A Minimal Jenkins Pipeline Example

Below is a learning-oriented, minimal Pipeline structure example to help you establish a basic understanding.

    pipeline {
        agent any

        stages {
            stage('Checkout') {
                steps {
                    echo 'Pull code'
                }
            }

            stage('Build') {
                steps {
                    echo 'Execute build'
                }
            }

            stage('Image Build') {
                steps {
                    echo 'Build image'
                }
            }

            stage('Push') {
                steps {
                    echo 'Push image to Harbor'
                }
            }

            stage('Deploy') {
                steps {
                    echo 'Deploy to Kubernetes'
                }
            }
        }
    }

This example has no actual commands, but it can first show you:

- Pipeline is divided into stages
- Each stage takes on a category of responsibilities
- The overall execution order is very clear

## Translating the Above Pipeline into Plain Language

It essentially says:

1. First pull code
2. Then build the program
3. Then build the image
4. Then push to Harbor
5. Then deploy to K8s

In other words, Pipeline isn't a mysterious thing; it's just writing the manual operation process as an automated script.

## A More Practical Jenkins Pipeline Approach

Below is a common approach for Java applications to K8s, but it remains learning-oriented and doesn't pursue complex syntax.

### Stage 1: Pull Code

Jenkins gets the code from the Git repository.

---

### Stage 2: Maven Package

Execute:

    mvn clean package -DskipTests

To get the jar package.

---

### Stage 3: Build Image

Execute:

    docker build -t harbor.example.com/demo/demo-app:${BUILD_NUMBER} .

Here `${BUILD_NUMBER}` can be understood as Jenkins's current build number, commonly used to generate a unique tag.

### Phase 4: Log in to Harbor and Push the Image

Execute:

    docker login harbor.example.com
    docker push harbor.example.com/demo/demo-app:${BUILD_NUMBER}

---

### Phase 5: Deploy to K8s

There are two simple approaches, for example.

#### Approach A: Directly Modify the Deployment Image

    kubectl set image deployment/demo-app demo-app=harbor.example.com/demo/demo-app:${BUILD_NUMBER} -n test

#### Approach B: Re-apply YAML

First update the image field in the YAML, then:

    kubectl apply -f deployment.yaml

---

### Phase 6: Check Deployment Results

Execute:

    kubectl rollout status deployment/demo-app -n test

At this point, the minimal "Jenkins → Harbor → K8s" mainline is connected.

## Three Common Implementation Approaches for Jenkins to K8s Deployment

This point is very important, as you will encounter it repeatedly in interviews, work, and learning.

---

## Approach 1: Jenkins + Docker + kubectl

This is the most intuitive and easiest to understand approach.

### Process

1. Jenkins pulls code
2. Jenkins packages
3. Jenkins builds the image
4. Jenkins pushes to Harbor
5. Jenkins directly executes kubectl to update the cluster

### Advantages

- Simple and direct
- Easy to understand
- Suitable for learning and early-stage projects

### Disadvantages

- YAML management can become messy
- Multi-environment maintenance is troublesome
- Weak configuration reuse capability
- Cluster access credentials management requires caution

Suitable for establishing mainline awareness in the early learning phase.

---

## Approach 2: Jenkins + Docker + Helm

This is a more common and more engineering approach.

### Process

1. Jenkins pulls code
2. Jenkins builds artifacts
3. Jenkins builds and pushes the image
4. Jenkins executes `helm upgrade --install`
5. Helm renders and deploys to K8s based on values

### Advantages

- Stronger templating capabilities
- Easier to maintain across multiple environments
- Clearer when managing resources manually

### Disadvantages

- Slightly higher learning cost
- Requires understanding of Charts and values

This is typically the mainstream approach for many enterprises when combining Jenkins + K8s.

---

## Approach 3: Jenkins + GitOps Repository + Argo CD

This is a more modern approach.

### Process

1. Jenkins pulls code and builds the image
2. Jenkins pushes to Harbor
3. Jenkins modifies the image tag in the GitOps repository
4. Jenkins commits to Git
5. Argo CD detects Git changes and synchronizes to the cluster

### Advantages

- Clearer cluster state
- More GitOps compliant
- Better audit, rollback, and change tracking

### Disadvantages

- Longer chain
- More systems involved
- Higher understanding cost

Currently, you don't need to make this the mainline, but know it's a common evolution direction.

## How Does Jenkins Access Kubernetes Clusters?

This is a critical point in actual implementation.

For Jenkins to deploy to K8s, it must first be able to access the cluster.

The most common approach is typically:

### 1) Use kubeconfig

Provide the available kubeconfig to Jenkins, and use:

- `kubectl`
- `helm`

This allows Jenkins to know which cluster to connect to and what identity to use.

---

### 2) Pre-install kubectl / helm on Jenkins Nodes

This means the node running the Pipeline must have these tools.

Otherwise, even if the pipeline script is written, it won't execute.

---

### 3) Manage authentication information via the credential system

For example:

- kubeconfig as Jenkins credentials
- Harbor username/password as Jenkins credentials
- Git credentials as Jenkins credentials

This is much safer than writing passwords directly in scripts.

## Why Jenkins Credentials Are So Important

Jenkins automation deployment typically involves multiple types of authentication:

- Git repository authentication
- Harbor authentication
- Kubernetes cluster authentication
- Helm repository authentication
- Notification system webhook, etc

If these are randomly written in the Pipeline, there will be significant risks.

Therefore, you will increasingly value Jenkins's credential management capabilities in the future.

At this stage, just remember one sentence:

> Never write passwords in Jenkins Pipeline unless absolutely necessary.

## What Are the Most Common Failure Points When Jenkins Deploys to K8s?

This section is very practical and very useful for troubleshooting later.

---

## First Category: Code and Build Layer Failures

For example:

- Git pull code fails
- Branch does not exist
- Maven dependency pull fails
- Compilation fails
- Test fails

---

## Second Category: Image Build Layer Failures

For example:

- Dockerfile has errors
- JAR package path is incorrect
- Build context is wrong
- Docker daemon is unavailable

---

## Third Category: Harbor Push Layer Failures

For example:

- docker login fails
- Robot account has insufficient permissions
- Harbor certificate is not trusted
- Tag is not standardized
- Push to the wrong project

---

## Fourth Category: K8s Deployment Layer Failures

For example:

- kubeconfig error
- kubectl cannot connect to the cluster
- namespace error
- RBAC permissions are insufficient
- Helm values error
- YAML rendering fails

---

## Fifth Category: Post-deployment Runtime Failures

For example:

- Nodes cannot pull the image
- Pod startup fails
- Readiness Probe fails
- ConfigMap / Secret configuration is incorrect
- Image version is updated, but the business still has issues

So, when you see "Jenkins deployment fails", don't just focus on Jenkins itself.

Often, the real problem is:

- The image
- Harbor
- kubeconfig
- K8s configuration
- Pod startup logic

## A Critical Understanding: Jenkins Is Just an Orchestration Tool, Not a Universal Executor

This understanding is very important.

Jenkins's main responsibilities are:

- Trigger actions in sequence
- Connect different systems
- Manage the process
- Aggregate results

But the actual execution is often done by these components:

- Git repository provides code
- Maven / Gradle handles the build
- Docker handles the image
- Harbor stores the image
- kubectl / Helm handles deployment
- Kubernetes runs the application

So you shouldn't think of Jenkins as a "do-it-all" tool.

More accurately:

> Jenkins is the orchestrator of the delivery pipeline.

## What's the most common misconception when learning Jenkins Pipeline

### Misconception 1: Memorizing syntax only

Many people start by memorizing:

- `pipeline {}`
- `stage {}`
- `steps {}`
- `script {}`

These are certainly important to know, but more importantly, you should first understand:

What problem each stage solves in the delivery chain.

---

### Misconception 2: Viewing Jenkins as the publishing system itself

Jenkins is just one entry point of the publishing process, not the entire delivery system.

---

### Misconception 3: Focusing only on publishing commands, not credentials and permissions

In actual production, authentication and permissions are often the most common sources of issues.

---

### Misconception 4: Assuming business success just because the pipeline succeeds

This is extremely dangerous.

Jenkins success only means "the command-level execution completed", not that the business is necessarily normal.

## What experiment is suitable for you as a learner now

This article is very suitable for doing a "minimum Jenkins delivery pipeline experiment".

## Recommended experiment: Manually simulate a Jenkins Pipeline workflow

If you don't have a Jenkins environment yet, you can first manually simulate its stage logic.

### Experiment objective

Manually complete the following stages according to Jenkins Pipeline logic:

Code → Build → Image → Harbor → K8s

### Recommended steps

#### Step 1: Build Java project

Execute:

    mvn clean package -DskipTests

Confirm the generation of a jar package.

#### Step 2: Build image

Execute:

    docker build -t harbor.example.com/demo/demo-app:v1 .

#### Step 3: Push to Harbor

Execute:

    docker login harbor.example.com
    docker push harbor.example.com/demo/demo-app:v1

#### Step 4: Deploy to K8s

Execute:

    kubectl apply -f deployment.yaml

Or:

    kubectl set image deployment/demo-app demo-app=harbor.example.com/demo/demo-app:v1 -n test

#### Step 5: Check deployment status

Execute:

    kubectl rollout status deployment/demo-app -n test

### The significance of this experiment

It allows you to first understand the essence of Jenkins Pipeline through "manual stages":

**Jenkins is simply automating these manual commands in a workflow.**

You only understand Jenkinsfile properly after manually running through it yourself.

## Key judgments to establish at this stage

### Judgment 1: Jenkins' core value is workflow orchestration

Not single-point tool capabilities, but connecting build, image, repository, and deployment into a chain.

### Judgment 2: Pipeline is "defining the publishing workflow with code"

Its core significance lies in maintainability, traceability, and reusability.

### Judgment 3: Jenkins to K8s publishing isn't just one way

At least understand:

- kubectl direct deployment
- Helm deployment
- GitOps indirect deployment

### Judgment 4: Jenkins publishing to K8s requires authentication and toolchain completeness

At least have:

- Git access capability
- Harbor credentials
- kubeconfig
- kubectl / helm tools

### Judgment 5: Jenkins success doesn't equal business success

Must also check:

- rollout status
- Pod status
- service accessibility
- logs and probes
- actual business validation

## One-sentence summary

**The essence of Jenkins Pipeline is to solidify the entire manual process of building, creating images, pushing to Harbor, and deploying to Kubernetes into a repeatable automated delivery workflow.**

## Operational extension understanding

From an operations and platform perspective, Jenkins is very much like a "bridge between history and reality".

Because many enterprises' delivery chains don't start with:

- GitOps
- Argo CD
- cloud-native platforms
- standardized delivery systems

But rather start with:

- Shell scripts
- SSH deployment
- Java package deployment
- VM deployment
- manual operations

Then gradually evolve to:

- Docker images
- Harbor
- K8s
- Helm
- GitOps

During this evolution, Jenkins is very common because it's flexible enough to accommodate many transitional phase needs.

So when you learn Jenkins later, you're not just learning a tool, but understanding:

**How traditional delivery systems evolved step by step to containerized and Kubernetes deployment systems.**

## Recommended learning path

Suggested to continue learning in this order:

1. 01-Why K8s needs CI-CD: From manual deployment to cloud-native application delivery
2. 02-Overall CI-CD workflow: Code, build, image, repository, and cluster deployment panorama
3. 03-Container image building basics: From Java programs to Docker images
4. 04-Harbor basics: Projects, repositories, tags, and Robot accounts
5. 05-GitLab CI basics: .gitlab-ci.yml, Stages, Jobs, and variables
6. 06-Jenkins Pipeline to K8s deployment: From traditional pipelines to containerized delivery
7. 07-K8s deployment basics: Deployments, image updates, and rolling update mechanisms
8. 08-Helm basics: Charts, Values, and application template management
9. 09-CI-CD deploymentActual: From building images to deploying to Kubernetes
10. 10-Deployment verification, rollback, and common issue troubleshooting: From successful deployment to stable delivery

## Tomorrow's plan

Next article will enter:

07-K8s deployment basics: Deployments, image updates, and rolling update mechanisms

Next article will focus on clarifying:

- Why Deployment is the most common application deployment object
- Why modifying an image triggers a rolling update
- The relationship between ReplicaSet and Pod during updates
- `kubectl rollout status`, `history`, `undo` respectively check what
- Why "Pod Running" still doesn't equal "business normal"