# 05-GitLab CI Basics: .gitlab-ci.yml, Stage, Job, and Variables

## Documentation Notes

This is the 5th note in the 08-CI-CD learning path.

The previous notes have gradually broken down the manual deployment pipeline step by step:

1. Manually build and deploy a minimal application to K8s
2. Break down the deployment process into fixed stages
3. Understand image building and image Tag design
4. Understand projects, repositories, tags, and Robot accounts in Harbor

This note marks the beginning of the first form of automation:

**GitLab CI.**

This note doesn't require you to write a complete pipeline from scratch, nor does it require you to fully connect all environments at once.  
The goals of this note are:

- Map the manual actions you've already done to GitLab CI
- Understand what `.gitlab-ci.yml` is actually controlling
- Understand what Stage, Job, and variables solve
- Use minimal examples to understand "which steps are automated by automation"

This note continues to align with the current learning path, assuming the following environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #GitLabCI #gitlab-ci.yml #Stage #Job #Variables #Harbor #AutomationOfReleases #I'llTakeYourNotes.

## Learning Objectives

After completing this note, you should understand:

1. Where GitLab CI fits in the entire release chain
2. The role of `.gitlab-ci.yml`
3. The difference between Stage and Job
4. Why variables are needed
5. How to map the manual actions you've done before to GitLab CI
6. How to write a minimal GitLab CI configuration skeleton
7. How to explain which steps will be automated by GitLab CI in the future

## First, Put GitLab CI Back into the Learned Chain

Previously, you've completed this chain through manual experiments:

Application content  
→ Image building  
→ Image storage  
→ Cluster configuration  
→ Cluster deployment  
→ Deployment verification

Now, don't rush to learn new syntax. Instead, ask this question first:

**Which steps will GitLab CI automate for you?**

In the current learning phase, the most common automation is the first half:

1. Pull code
2. Build application
3. Build image
4. Push to Harbor

Further automation can also include:

5. Update configuration
6. Deploy to K8s
7. Perform basic verification

So you can initially understand GitLab CI as:

**Putting the fixed commands you've manually executed into an automated execution framework.**

## Part One: Understand What GitLab CI Actually Does

### Core Understanding

GitLab CI itself is not code, nor is it Harbor, nor is it Kubernetes.  
Its role is more like an automation execution hub:

- It detects code changes
- Executes commands according to your rules
- Displays logs and results
- Determines whether the automation process is successful or failed

### Understanding to Establish Now

You've manually done these actions before:

- `docker build`
- `docker tag`
- `docker push`
- `kubectl set image`
- `kubectl rollout status`

What GitLab CI does is essentially:

**Automatically run these manual actions in sequence.**

## Part Two: What is `.gitlab-ci.yml`

`.gitlab-ci.yml` is the core configuration file of GitLab CI.

You can initially understand it as:

**The instruction manual for this automation pipeline.**

It's usually placed in the root directory of the code repository, telling GitLab:

- What stages are in this pipeline
- What tasks are in each stage
- What commands each task executes
- What variables are used
- When each task executes

### Understanding to Establish Now

Without `.gitlab-ci.yml`, GitLab doesn't know what automation you want.  
So it's not an optional small configuration, but rather:

**The entry file for the automation deployment process.**

## Part Three: What are Stage and Job

These are the two most important basic concepts in GitLab CI.

### 1) Stage

Stage can be understood as:

**The major stages of the pipeline.**

For example:

- build
- image
- deploy

It solves:

- What to do first
- What to do next
- How to layer the process order

### 2) Job

Job can be understood as:

**A specific task within a stage.**

For example:

- `build_app`
- `build_image`
- `push_harbor`
- `deploy_test`

It solves:

- What specific commands to execute
- What this task is called
- Which stage it belongs to

### Understanding to Establish Now

You can initially remember:

- Stage is "major steps"
- Job is "specific actions within major steps"

## Part Four: Map Your Manual Actions to Stage and Job

Bring back the manual actions you've done before.

### Manual Actions You've Done

1. Modify `index.html`
2. `docker build`
3. `docker tag`
4. `docker push`
5. `kubectl set image`
6. `kubectl rollout status`
7. `wget` Verify page results

### These Can Be Mapped to the Following Stages

- build
- image
- deploy
- verify

### These Can Be Mapped to the Following Jobs

- `build_image`
- `push_image`
- `deploy_test`
- `verify_test`

In other words, GitLab CI hasn't invented new deployment actions. It's simply organizing the actions you've already done manually.

## Part Five: Why Variables Exist

If you've manually executed all commands before, you'll know that many elements appear repeatedly, such as:

- Harbor address
- Project name
- Image name
- Image tag
- namespace
- Deployment name

If all these are hard-coded in the pipeline, you'll quickly face problems:

- Changing one place requires changing everywhere
- dev / test / prod are hard to distinguish
- Prone to errors
- Poor maintainability

So the core role of variables is:

**Extracting easily changing, repeated values for unified management.**

### Understanding to Establish Now

Variables aren't for "advanced syntax," but for:

- Reducing repetition
- Reducing errors
- Facilitating environment switching
- Unifying management of image tags, Harbor addresses, etc.

## Part Six: Write a Minimal `.gitlab-ci.yml` Skeleton First

This section doesn't rush to write complex commands. Only establish the structural sense first.

Look at a minimal skeleton:

    stages:
      - image
      - deploy

    build_image:
      stage: image
      script:
        - echo "build image"

deploy_test:
  stage: deploy
  script:
    - echo "deploy to test"

### What This Configuration Expresses

1. The pipeline has two stages:
   - image
   - deploy

2. There are two Jobs:
   - `build_image`
   - `deploy_test`

3. `build_image` belongs to `image` stage
4. `deploy_test` belongs to `deploy` stage
5. The actual commands executed by each Job are written in `script`

### Current Understanding to Establish

`.gitlab-ci.yml`'s core is not "remembering YAML format", but understanding this structure:

- Define stages first
- Then define tasks
- Each task belongs to a stage
- Each task executes a group of commands

## Part 7: Put Your Previously Executed Commands into Jobs

Now start putting your previously executed manual actions into GitLab CI thinking.

### Manual Image Build Actions You've Done Before

Example:

    docker build -t manual-web:v3 .

If placed into GitLab CI thinking, it becomes:

    build_image:
      stage: image
      script:
        - docker build -t manual-web:v3 .

At this point you should realize:

**The script in a Job is essentially the commands you previously typed manually.**

### Harbor Push Actions You've Done Before

Example:

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301
    docker push harbor.example.com/test/manual-web:dev-c1d2e3f-301

If placed into GitLab CI thinking, it can become:

    push_image:
      stage: image
      script:
        - docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301
        - docker push harbor.example.com/test/manual-web:dev-c1d2e3f-301

This shows:

- GitLab CI doesn't change the commands themselves
- It just organizes these commands into the pipeline

## Part 8: Start Introducing Variables to Extract Hardcoded Information

If you hardcode Harbor address, image name, and Tag, it will be hard to maintain later.

Here's a more reasonable structure:

    variables:
      HARBOR_REGISTRY: "harbor.example.com"
      HARBOR_PROJECT: "test"
      IMAGE_NAME: "manual-web"
      IMAGE_TAG: "dev-c1d2e3f-301"

    build_image:
      stage: image
      script:
        - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        - docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
        - docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}

### Current Understanding to Establish

Variables extract these "values that often repeat":

- Harbor address
- Harbor project
- Image name
- Tag

Later you just need to change the variables, without modifying the command bodies everywhere.

## Part 9: Combine with Your Current Learning Environment, Do a "Manual Simulation of GitLab CI" Exercise

You may not have a ready GitLab CI environment now, which is fine.  
This section focuses on a more important task first:

**Treat each GitLab CI Job as a "scripted manual action" to understand.**

### Exercise Objective

Re-do the following 3 groups of actions manually, and mark which GitLab CI Job they will belong to.

#### Action 1: Build Image

Execute:

    docker build -t manual-web:v5 .

Corresponding understanding:

- Will belong to `build_image` type of Job

#### Action 2: Push to Harbor

Execute:

    docker tag manual-web:v5 your-Harbor-domain/test/manual-web:dev-test001-501
    docker push your-Harbor-domain/test/manual-web:dev-test001-501

Corresponding understanding:

- Will belong to `push_image` type of Job

#### Action 3: Update Deployment

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-test001-501

Corresponding understanding:

- Will belong to `deploy_test` type of Job

### Current Understanding to Establish

Here you're not repeating old experiments, but establishing a mapping relationship:

- Which Job this manual command will belong to
- Which Stage this Job should be placed in
- What content should be written as variables

After completing this step, you'll no longer find GitLab CI abstract.

## Part 10: Now Write a Minimal GitLab CI Example Closer to Your Current Learning Focus

The following configuration doesn't require you to run it immediately, just understand the structure.

    stages:
      - image
      - deploy

    variables:
      HARBOR_REGISTRY: "harbor.example.com"
      HARBOR_PROJECT: "test"
      IMAGE_NAME: "manual-web"
      IMAGE_TAG: "dev-c1d2e3f-301"
      NAMESPACE: "test"
      DEPLOY_NAME: "manual-web"

```markdown
build_and_push_image:
  stage: image
  script:
    - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
    - docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
    - docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}

deploy_test:
  stage: deploy
  script:
    - kubectl -n ${NAMESPACE} set image deployment/${DEPLOY_NAME} ${IMAGE_NAME}=${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
    - kubectl -n ${NAMESPACE} rollout status deployment/${DEPLOY_NAME}

### How to Read This Configuration

Don't rush to memorize syntax details yet—just look at the structure first:

#### Stage

- `image`
- `deploy`

#### Job

- `build_and_push_image`
- `deploy_test`

#### Variables

- Harbor Address
- Project Name
- Image Name
- Tag
- Namespace
- Deployment Name

#### script

Everything inside is the exact command you've manually typed before.

### Current Understanding to Establish

At this point, you should be able to naturally say:

**GitLab CI isn't doing any new mysterious things—it's simply putting the build, push, and deploy commands you've manually executed into an automation framework.**

## Part 11: Why GitLab CI Must Have a Runner

This section won't discuss implementation details yet—just the understanding you must establish now.

The GitLab CI page itself will not execute `docker build`.  
The real executor of these commands is the Runner.

You can initially think of the Runner as:

**The machine that executes the pipeline.**

Without a Runner, you'd encounter this situation:

- GitLab sees `.gitlab-ci.yml`
- Knows which Jobs to run
- But has no real executor
- So the Jobs can't run

### Current Understanding to Establish

When learning GitLab CI later, remember:

- `.gitlab-ci.yml` defines the workflow
- Runner executes the commands

## Part 12: Reconnecting This with Previous Sections

Now merge all previous sections together, and you should understand it this way:

### Part 1

You manually completed a full release.

### Part 2

You split the release into several fixed stages.

### Part 3

You learned image building and production-style Tag conventions.

### Part 4

You pushed the image to Harbor and understood the repository structure.

### Part 5

You started mapping all your manual actions to GitLab CI:

- Use `.gitlab-ci.yml` to define the workflow
- Use Stage to manage order
- Use Job to place actions
- Use variables to reduce repetition
- Use Runner to actually execute commands

At this point, GitLab CI is truly connected to your learningMain.

## Part 13: This Section's Practice Exercise

### Exercise 1: Manually Write Your Own Minimum `.gitlab-ci.yml`

Requirements must include at least:

- 2 Stages
- 2 Jobs
- 3 Variables

Suggested structure:

- One Job for image build + push
- One Job for Deployment update

You don't need it to run yet—just write the structure.

### Exercise 2: Classify Your Previous 6 Commands into Jobs

For example:

- Which commands belong to `build_and_push_image`
- Which belong to `deploy_test`

### Exercise 3: Answer These 4 Questions Yourself

1. What does `.gitlab-ci.yml` do in the entire workflow
2. What's the difference between Stage and Job
3. Why do variables exist
4. Why is Runner a necessary role in GitLab CI

If you can explain these 4 questions yourself, you've mastered this section.

## Content to Be Able to Explain After This Section

After completing this section, you should be able to clearly explain this:

GitLab CI's purpose is to automate the build, push, deploy, and other manual actions.  
`.gitlab-ci.yml` is the pipeline definition file, which organizes large stages through Stage, defines specific tasks through Job, and unifies management of Harbor address, image name, Tag, and other easily changing values through variables.  
The script in Job is essentially the exact commands you've manually executed before.  
The real executor of these commands isn't the GitLab page itself, but the Runner.  
So GitLab CI's core isn't inventing new release actions, but standardizing, automating, and making repeatable the existing release actions.

## Common Issues and Troubleshooting Directions

### Issue 1: Why does `.gitlab-ci.yml` look like YAML but is hard to understand

Because without prior manual release experience, you might see it as pure syntax.  
Now you should reverse this understanding:

- It's just moving the commands you've already executed into it

### Issue 2: Are more variables always better

No.  
Variables aim to extract repeated and easily changing values, not to make everything a variable.

### Issue 3: Why do I understand `.gitlab-ci.yml` but still can't run it

Because running requires:

- GitLab repository
- Runner
- Credentials
- Docker / kubectl / Harbor and other environment conditions

These are content that will be continued in subsequent sections.

## Key Points Mastered in This Section

After completing this section, you should master:

1. GitLab CI's position in the entire workflow
2. The role of `.gitlab-ci.yml`
3. The difference between Stage and Job
4. The value of variables
5. The role of Runner
6. How to map your manual commands to GitLab CI

## One-Sentence Summary

The essence of GitLab CI is using `.gitlab-ci.yml` to organize the fixed actions of building, pushing images, and updating deployments you've manually completed into an automated pipeline supported by Stage, Job, variables, and Runner.

## Next Section

Next section will enter:

06-Jenkins Pipeline to K8s Deployment: From Traditional Pipeline to Containerized Delivery

The next section will continue using the same learning approach, mapping the actions you've manually performed to the Jenkins Pipeline perspective, with a focus on understanding: /think
```

- Comparison of similarities and differences between Jenkins and GitLab CI  
- What does Pipeline and Jenkinsfile correspond to in the current mainline?  
- Why do many enterprises gradually transition from Jenkins to a more comprehensive cloud-native delivery system?