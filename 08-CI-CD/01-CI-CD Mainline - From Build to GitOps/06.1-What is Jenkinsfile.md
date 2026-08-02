# What is Jenkinsfile?

Jenkinsfile is a text file used in Jenkins to define an automated build and deployment process. It contains a sequence of commands that Jenkins executes to perform various tasks, such as pulling code from a repository, compiling the source code, building Docker images, pushing the images to a repository, and deploying the application to Kubernetes.

Typically, Jenkinsfile is written in Groovy, a high-level scripting language designed for Jenkins. It allows users to specify the build process in a structured and reusable way, making it easier to maintain and update the deployment pipeline over time.

In simple terms, Jenkinsfile is like a script that tells Jenkins what to do step by step to complete the entire deployment process.If we say that a Pipeline is the “release process”, then a Jenkinsfile is:

**the definition document for this process.**

It is usually placed in the root directory of the code repository.

For example:

    Jenkinsfile

This way, the code and the pipeline definition can be managed together.

## A minimal Jenkins Pipeline example

Here’s a simplified example of a Pipeline structure to help you get started.

    pipeline {
        agent any

        stages {
            stage('Checkout') {
                steps {
                    echo 'Pull the code'
                }
            }

            stage('Build') {
                steps {
                    echo 'Execute building'
                }
            }

            stage('Image Build') {
                steps {
                    echo 'Build the image'
                }
            }

            stage('Push') {
                steps {
                    echo 'Push the image to Harbor'
                }
            }

            stage('Deploy') {
                steps {
                    echo 'Deploy to Kubernetes'
                }
            }
        }
    }

This example doesn’t contain any actual commands, but it shows you that:

- A Pipeline is divided into stages.
- Each stage performs a specific task.
- The overall execution order is clear.

## Putting the above Pipeline into plain language

Essentially, it means:

1. First, pull the code.
2. Then, build the program.
3. Next, build the image.
4. After that, push the image to Harbor.
5. Finally, deploy it to K8s.

In other words, a Pipeline isn’t something mysterious; it’s just an automated script that converts manual operations into a sequence of steps.

## A more practical Jenkins Pipeline approach

Here’s a common workflow for deploying a Java application to K8s, still kept simple for learning purposes.

### Step 1: Pull the code

Jenkins retrieves the current code from the Git repository.

---

### Step 2: Package with Maven

Run:

    mvn clean package -DskipTests

This creates a jar package.

---

### Step 3: Build the image

Run:

    docker build -t harbor.example.com/demo/demo-app:${BUILD_NUMBER} .

Here, `${BUILD_NUMBER}` represents Jenkins’ current build number, which is used to generate a unique tag for the image.

---

### Step 4: Log in to Harbor and push the image

Run:

    docker login harbor.example.com
    docker push harbor.example.com/demo/demo-app:${BUILD_NUMBER}

---

### Step 5: Deploy to K8s

There are two common approaches:

#### Approach A: Directly update the Deployment image

    kubectl set image deployment/demo-app demo-app=harbor.example.com/demo/demo-app:${BUILD_NUMBER} -n test

#### Approach B: Apply the updated YAML file again

First, update the `image` field in the YAML file, and then:

    kubectl apply -f deployment.yaml

---

### Step 6: Check the deployment result

Run:

    kubectl rollout status deployment/demo-app -n test

Now, you have a basic “Jenkins → Harbor → K8s” workflow.

## Three common approaches for Jenkins to deploy to K8s

This is an important point that you will encounter frequently in interviews, work, and further studies.

---

## Approach 1: Jenkins + docker + kubectl

This is the most straightforward and easy-to-understand approach.

### Process

1. Jenkins pulls the code.
2. Jenkins packages the code.
3. Jenkins builds the image.
4. Jenkins pushes the image to Harbor.
5. Jenkins directly uses `kubectl` to update the Kubernetes cluster.

### Advantages

- Simple and easy to understand.
- Suitable for learning and initial projects.

### Disadvantages

- YAML configuration can become cumbersome.
- Managing multiple environments can be challenging.
- Limited ability to reuse configurations.
- Careful management of cluster access credentials is required.

This approach is ideal for getting started with Jenkins and understanding the basic workflow.

---

## Approach 2: Jenkins + docker + Helm

This is a more common and more engineered approach.

### Process

1. Jenkins pulls the code.
2. Jenkins builds the application.
3. Jenkins builds and pushes the image.
4. Jenkins executes `helm upgrade --install`.
5. Helm uses the provided configuration files to deploy the application in K8s.

### Advantages

- Strong templating capabilities.
- Easier management of multiple environments.
- clearer resource management compared to manual YAML configuration.

### Disadvantages

- Higher learning curve.
- Requires understanding of Charts and values files.

This is often the preferred approach for many organizations when using Jenkins with Kubernetes.

---

## Approach 3: Jenkins + GitOps repository + Argo CD

This is a more modern approach.

### Process

1. Jenkins pulls the code and builds the image.
2. Jenkins pushes the image to Harbor.
3. Jenkins updates the image tag in the GitOps repository.
4. Jenkins commits the changes to Git.
5. Argo CD detects the changes and applies them to the Kubernetes cluster- `script {}`

Of course, these need to be understood, but more importantly, it's essential to first grasp what problem each phase solves within the delivery chain.

---

### Misconception 2: Treating Jenkins as the Publishing System Itself

Jenkins is just one of the entry points in the publishing process; it does not represent the entire delivery system.

---

### Misconception 3: Focusing Only on Publishing Commands, Ignoring Credentials and Permissions

In actual production environments, authentication and permissions are often where issues arise most frequently.

---

### Misconception 4: Assuming Business Success Just Because the Pipeline Successfully Completed

This is very dangerous. Jenkins' success only indicates that the commands were executed; it does not mean that the business is functioning properly.

## As a Learner, What Experiments Are Suitable for You Now?

This article is perfect for conducting a "minimal Jenkins publishing pipeline experiment."

## Recommended Experiment: Manually Simulate a Jenkins Pipeline

If you don't have a Jenkins environment yet, you can still manually simulate its phase logic.

### Experimental Objective

Following the approach of a Jenkins Pipeline, manually complete the following steps:

Code → Build → Image → Harbor → K8s

### Recommended Steps

#### Step 1: Build the Java Project

Execute:

    mvn clean package -DskipTests

Verify that a jar package has been generated.

#### Step 2: Build the Image

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

#### Step 5: Check the Publishing Status

Execute:

    kubectl rollout status deployment/demo-app -n test

### The Significance of This Experiment

It allows you to understand the essence of a Jenkins Pipeline from a "manual phase" perspective:

**Jenkins merely automates these manual commands in a sequential process.**

Only by performing these steps manually first can you truly grasp the meaning behind a Jenkinsfile when you encounter it later.

## Key Judgments You Should Make at This Stage

### Judgment 1: The Core Value of Jenkins Lies in Process Orchestration

It's not about the capabilities of individual tools, but about linking build, image creation, repository management, and publishing together.

### Judgment 2: Pipelines Are "Codes That Define Publishing Processes"

Their true significance lies in their maintainability, traceability, and reusability.

### Judgment 3: There Are Multiple Ways to Publish from Jenkins to K8s

At least understand the following methods:

- kubectl direct deployment
- Helm deployment
- GitOps indirect publishing

### Judgment 4: The Prerequisites for Jenkins to Publish to K8s Include Proper Authentication and Toolchains

You need at least:

- Git access capabilities
- Harbor credentials
- kubeconfig
- Tools like kubectl and helm

### Judgment 5: Jenkins' Success Does Not Necessarily Mean Business Success

You also need to check:

- The rollout status
- Pod status
- Service accessibility
- Logs and monitoring tools
- Actual business functionality

## In One Sentence

**The essence of a Jenkins Pipeline is to transform the manual processes of building, creating images, pushing to Harbor, and deploying to Kubernetes into a repeatable automated delivery process.**

## Additional Insights from an Operations Perspective

From an operations and platform standpoint, Jenkins functions much like a "bridge between the past and the present."

Many companies do not start with advanced systems like GitOps, Argo CD, cloud-native platforms, or standardized delivery pipelines. Instead, they begin with:

- Shell scripts
- SSH-based publishing
- Java package deployments
- Virtual machine management
- Manual operations

It is only over time that these systems evolve into Docker images, Harbor, K8s, Helm, and GitOps. During this transition, Jenkins plays a crucial role because it is flexible enough to accommodate various intermediate needs.

Therefore, learning Jenkins is not just about mastering a tool; it's also about understanding how traditional delivery systems have gradually evolved towards containerized and Kubernetes-based approaches.

## Recommended Learning Path

We suggest continuing with the following order:

1. 01-Why K8s Needs CI-CD: From Manual Publishing to Cloud-Native Application Delivery
2. 02-CI-CD Overall Chain: Code, Build, Image, Repository, and Cluster Deployment Overview
3. 03-Container Image Building Basics: From Java Programs to Docker Images
4. 04-Harbor Basics: Projects, Repositories, Tags, and Robot Accounts
5. 05-GitLab CI Basics:.gitlab-ci.yml, Stages, Jobs, and Variables
6.