# 14-GitLab Runner and Jenkins Agent Basics: How Do Pipeline Executors Work?

## Document Notes

This is the 14th note in the 08-CI-CD learning path.

The previous sections 01-13 have already run through a minimal deployment pipeline, and gradually filled in:

- Image building
- Image tagging
- Harbor repository
- GitLab CI
- Jenkins Pipeline
- K8s rolling update
- Helm
- Multi-environment deployment

In this section, we begin to fill in a very critical but often overlooked role:

**Pipeline executor.**

Many learners of GitLab CI or Jenkins tend to focus on:

- `.gitlab-ci.yml`
- Jenkinsfile
- stage
- job
- pipeline

These "definition layers".

But during actual execution, the commands are not executed by GitLab pages or Jenkins pages, but by:

- GitLab Runner
- Jenkins Agent

This article's focus is to clearly explain this execution layer and correspond it with the manual commands you've already done.

This article continues to align with the current experiment's main line and environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- The minimal application `manual-web` has been built, pushed, deployed, and verified

## Tags

#Kubernetes #CI-CD #GitLabRunner #JenkinsAgent #WaterlineExecutor #Harbor #Docker #kubectl #I'llTakeYourNotes.

## Learning Objectives

After completing this section, you should understand:

1. What GitLab Runner and Jenkins Agent are
2. Why pipelines must have an executor
3. That "pipeline definition" and "pipeline execution" are two distinct concepts
4. What tools are typically required on Runners/Agents
5. Use the commands you've already executed manually to understand the executor's role in actual deployment
6. Explain why without an executor, even a complete CI/CD configuration cannot run

## This Section's Experiment Main Line

This section does not require you to fully install GitLab Runner or Jenkins Agent and integrate them with the platform.  
At this stage, the focus is on three things:

1. First, separate the "definition layer" and "execution layer" for understanding
2. Use the commands you've already manually executed to reverse-engineer what the executor is doing
3. Conduct a "minimum local script execution experiment" to simulate the working mode of Runners/Agents

---

## Part One: First Answer a Core Question - Who Actually Executes the Pipeline Commands

You've already seen:

### In GitLab CI:

- `.gitlab-ci.yml`
- stage
- job
- script

### In Jenkins:

- Jenkinsfile
- Pipeline
- stage
- steps
- `sh '...'`

These things define:

- What to do
- What to do first
- What to do next

But they themselves do not actually execute commands.

For example, in GitLab CI, you write:

    script:
      - docker build -t manual-web:v10 .

Or in a Jenkinsfile:

    sh 'docker build -t manual-web:v10 .'

The actual executor of this command is not the web page, but a real execution environment.

This execution environment is:

- GitLab Runner
- Jenkins Agent

### Understanding to Establish at This Stage

Pipelines have two layers:

#### First Layer: Definition Layer

For example:

- `.gitlab-ci.yml`
- Jenkinsfile

#### Second Layer: Execution Layer

For example:

- GitLab Runner
- Jenkins Agent

All your future understanding of CI/CD must separate these two layers.

---

## Part Two: What Is GitLab Runner

You can initially understand GitLab Runner as:

**A dedicated work node for executing GitLab CI jobs.**

Its responsibilities are:

1. Receive jobs assigned by GitLab
2. Prepare the execution environment
3. Pull code
4. Execute the commands in `.gitlab-ci.yml`'s script
5. Return logs and results to GitLab

### A More Direct Understanding

GitLab Runner is like a "worker who takes orders and does the work".

GitLab itself is more like a "scheduling center":

- Detects code changes
- Reads `.gitlab-ci.yml`
- Generates a pipeline
- Decides which job to execute

Runner is more like the "worker who actually does the work":

- Actually runs the commands
- Actually builds
- Actually pushes
- Actually deploys

### Understanding to Establish at This Stage

Whether GitLab CI can run depends not only on whether `.gitlab-ci.yml` is written correctly, but also on:

- Whether there is a Runner
- Whether the Runner can accept tasks
- Whether the Runner has the required tools

---

## Part Three: What Is Jenkins Agent

You can initially understand Jenkins Agent as:

**A dedicated work node for executing Jenkins Pipeline stages/steps.**

Its responsibilities are similar to GitLab Runner, also:

1. Receive tasks assigned by Jenkins
2. Prepare the execution environment
3. Execute commands in the Jenkinsfile
4. Return logs and results

### A More Direct Understanding

Jenkins itself is like a "pipeline management center",  
Agent is more like the "actual machine doing the work".

So the command you write in a Jenkinsfile:

    sh 'docker build -t manual-web:v10 .'

Must ultimately be executed by some Agent.

### Understanding to Establish at This Stage

Jenkins Pipeline is not page magic.  
Without an Agent, even the most beautiful Jenkinsfile is just "written there".

---

## Part Four: Why Pipelines Must Have an Executor

This question must be thoroughly explained, as it helps you fundamentally understand the value of Runners/Agents.

Suppose you write a perfect `.gitlab-ci.yml`:

- Has build
- Has push
- Has deploy
- Has verify

But without a Runner, what happens?

The answer is:

- GitLab knows how the pipeline is defined
- But there is no place to actually execute the commands
- So the job will be pending or simply not run at all

Jenkins is the same.

The fundamental reason for the existence of an executor is:

> **CI/CD is not just about "describing the process," it must also have "machines to execute the process."**

### Understanding to Establish at This Stage

CI/CD configuration files solve "how to describe the automation process."  
Executors solve "who actually runs this process."

## Part Five: Reverse-Mapping Your Manual Commands to the Executor

This is the most critical understanding at this stage.

You've already manually performed these actions:

- `docker build`
- `docker tag`
- `docker push`
- `kubectl set image`
- `kubectl rollout status`
- `wget` Validate page results

Now view them from a different perspective:

### If these actions are delegated to GitLab Runner / Jenkins Agent in the future, they essentially execute these commands

For example:

#### 1) Build an image

    docker build -t manual-web:v10 .

This is what the executor is doing.

#### 2) Push to Harbor

    docker tag manual-web:v10 harbor.example.com/test/manual-web:v10
    docker push harbor.example.com/test/manual-web:v10

This is also what the executor is doing.

#### 3) Deploy to K8s

    kubectl -n test set image deployment/manual-web manual-web=harbor.example.com/test/manual-web:v10

This is the same action the executor is performing.

#### 4) Check rollout status

    kubectl -n test rollout status deployment/manual-web

Still, this is what the executor is doing.

### Understanding to Establish at This Stage

The executor isn't inventing new actions; it's simply moving the commands you've already manually executed into an automated workflow.

---

## Part Six: Tools Typically Required on Runner / Agent

This is a very practical section.

Many people write CI/CD configurations, but they fail to run, often not due to syntax, but because:

**The executor environment lacks necessary tools.**

### Combining with Your Current Learning Path, the Executor Typically Needs These Capabilities

#### 1) Git Capabilities

Because it usually needs to pull code first.

#### 2) Docker or Other Image Building Capabilities

Because it needs to perform:

- `docker build`
- `docker tag`
- `docker push`

#### 3) Harbor Login Capabilities

Because it needs to execute:

- `docker login`
- `docker push`

#### 4) kubectl

If the pipeline needs to deploy to K8s, it must have:

- `kubectl`
- Corresponding kubeconfig or cluster access permissions

#### 5) Helm

If the pipeline uses Helm for deployment, it needs:

- `helm`

### Understanding to Establish at This Stage

Just because you can write `.gitlab-ci.yml` / Jenkinsfile doesn't mean the pipeline will run successfully.  
The environment where the executor resides must have the actual toolchain to perform tasks.

---

## Part Seven: Why Some Pipelines Build Successfully but Deploy Fail

This is a common issue many people encounter.

The usual cause isn't "the pipeline mysteriously broke," but rather:

- The executor can build
- But lacks deploy capabilities

For example:

### Case 1: Can build, cannot push

Possible reasons:

- Runner / Agent has Docker
- But lacks Harbor credentials
- Or Harbor domain certificate is not trusted

### Case 2: Can push, cannot deploy

Possible reasons:

- Runner / Agent has Docker
- Can push to Harbor
- But lacks `kubectl`
- Or no kubeconfig
- Or insufficient permissions

### Case 3: Can deploy, but verification fails

Possible reasons:

- Deployment command was issued
- But Pod can't pull the image
- Or rollout is stuck
- Or business results are incorrect

### Understanding to Establish at This Stage

When encountering pipeline issues in the future, ask yourself:

> Is it a definition layer problem, or does the executor environment lack capabilities?

---

## Part Eight: Commonalities Between GitLab Runner and Jenkins Agent

At this stage, don't rush to distinguish too finely; focus on commonalities first.

### Commonality 1: Both are actual places where commands are executed

### Commonality 2: Both need to receive tasks assigned by the platform

### Commonality 3: Both depend on local tooling environments

### Commonality 4: Both may fail due to missing tools, credentials, or permissions

### Commonality 5: Both are merely executors, not the business logic itself

### Understanding to Establish at This Stage

Although Runner and Agent belong to different platforms, in the current learning context, they play highly similar roles:

**They are both "machines that help execute pipeline commands."**

---

## Part Nine: Differences Between GitLab Runner and Jenkins Agent - How to Understand at This Stage

At this stage, don't delve into plugin systems or architecture details; just establish the most practical distinctions.

### GitLab Runner

Closely aligned with GitLab CI:

- Accepts `.gitlab-ci.yml`
- Naturally integrates more tightly with GitLab repository events

### Jenkins Agent

Closely aligned with Jenkins Pipeline:

- Accepts Jenkinsfile
- More like an execution node of the Jenkins automation platform

### Understanding to Establish at This Stage

For you now, the focus isn't on determining which is more advanced,  
but rather to know:

- GitLab CI relies on Runner for implementation
- Jenkins Pipeline relies on Agent for implementation

---

## Part Ten: Create a "Minimum Executor Simulation Experiment" Locally

This section is very important because it transforms Runner / Agent from abstract concepts into actionable understanding.

At this stage, you don't need to connect to GitLab or Jenkins; just simulate the executor's working mode first.

### Step 1: Create a Script Locally

Enter a new directory:

    mkdir -p ~/08-ci-cd/14-runner-agent-lab
    cd ~/08-ci-cd/14-runner-agent-lab

Create the script:

    cat > run-pipeline.sh <<'EOF'
    #!/usr/bin/env bash
    set -e

    echo "== Build =="
    cd ~/08-ci-cd/01-manual-release
    docker build -t manual-web:runner-test .

    echo "== Tag =="
    docker tag manual-web:runner-test Yours.HarborDomain Name/test/manual-web:runner-test

    echo "== Push =="
    docker push Yours.HarborDomain Name/test/manual-web:runner-test

```markdown
echo "== Deploy =="
kubectl -n test set image deployment/manual-web manual-web=Yours.HarborDomain Name/test/manual-web:runner-test

echo "== Rollout =="
kubectl -n test rollout status deployment/manual-web

echo "== Done =="
EOF

Give the script execute permissions:

    chmod +x run-pipeline.sh

Note: Replace `Yours.HarborDomain Name` with the actual value.

### Step 2: Execute the script

    ./run-pipeline.sh

### Current understanding to establish

This step, although not a GitLab Runner or Jenkins Agent,  
is very similar to what an executor does:

- Execute a sequence of commands in order
- Any failure in any step affects the entire process
- Success indicates the machine has the required capabilities for the execution chain

You can initially understand this machine running the script as a "local simulation executor".

---

## Part 11: Use this simulation experiment to reverse-engineer the requirements of an executor

If the script above fails to execute, you can directly see the most common constraints of an executor.

### If build fails

Indicates:

- Dockerfile has issues
- Content is not ready
- Or the machine's Docker capabilities are problematic

### If push fails

Indicates:

- Harbor login has issues
- Repository address is incorrect
- Or certificate/permission issues

### If deploy fails

Indicates:

- `kubectl` is unavailable
- kubeconfig is unavailable
- Namespace or resource object issues

### If rollout fails

Indicates:

- The deployment action was sent
- But Pod layer issues may exist
- Need to troubleshoot at the K8s layer

### Current understanding to establish

An executor is not an "abstract role"—it's just a real machine doing the work.  
It must have:

- Docker
- Harbor credentials
- kubectl
- kubeconfig
- Helm

These will directly determine whether the pipeline can be implemented.

---

## Part 12: Why having more executor capabilities doesn't necessarily mean better

This point requires establishing a boundary awareness in advance.

On the surface, it seems better for an executor to be able to do everything.  
But in reality, it's not.

### Problem 1: Potential over-permission

For example:

- Can push to Harbor freely
- Can deploy to all clusters
- Can modify all namespaces

This poses significant risks in production.

### Problem 2: Unclear responsibilities

Some executors are suitable for:

- Only doing build

Others are suitable for:

- Only doing deploy

Mixing them together may be convenient, but it's not necessarily reasonable.

### Current understanding to establish

At this stage, you first need to understand "an executor must have capabilities".  
Later, when entering the security section, you'll further understand:

**An executor isn't just required to be able to work—it should also have minimal permissions.**

---

## Part 13: What a minimal executor must have based on your current environment

From your current learning path, a minimal usable executor must at least have:

1. Docker
2. Harbor login capability
3. kubectl
4. Access capability to K8s cluster
5. Basic Shell environment

If you later want to use Helm, add:

6. Helm

### Current understanding to establish

In the future, when you look at GitLab Runner / Jenkins Agent, don't just think of them as "platform components",  
also consider:

- Whether the machine they're on has these minimal capabilities

---

## Part 14: This section's exercise

### Exercise 1: Run a local simulation executor script

Requirements:

- The script must include at least four sections: build / push / deploy / rollout
- It should be able to run a minimal deployment

### Exercise 2: Intentionally remove the deploy section from the script, only do build + push

Then answer yourself:

- This indicates the executor currently has which part of the capabilities
- What capabilities are still missing

### Exercise 3: Answer the following 5 questions yourself

1. What is GitLab Runner?
2. What is Jenkins Agent?
3. Why must a pipeline have an executor?
4. Why does having a correctly written configuration file not guarantee a pipeline will run?
5. What minimal capabilities must an executor have at least?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content you should be able to explain after this section

After completing this section, it's recommended you can clearly explain the following:

GitLab Runner and Jenkins Agent are essentially both pipeline executors.  
`.gitlab-ci.yml` and Jenkinsfile define the process, but the actual execution of build, push, deploy commands is done by the environment where Runner or Agent resides.  
So whether a pipeline can be implemented isn't just about whether the definition files are correct—it also depends on whether the executor has the actual toolchain capabilities like Docker, Harbor login, kubectl, and cluster access permissions.  
In this learning phase, you can first simulate the executor's working mode locally to more intuitively understand the execution layer of CI/CD.

## Common Issues and Troubleshooting Directions

### Issue 1: Why do I feel uneasy even though I can write GitLab CI / Jenkinsfile

Because you may only understand the definition layer and haven't truly established a sense of the execution layer.

### Issue 2: Is Runner / Agent just a regular machine

It can be understood as such essentially, but it usually also registers, schedules, and binds with the platform through permissions and task binding.

### Issue 3: Why might the same pipeline produce different results on different executors

Because the execution environment may differ:

- Different Docker versions
- Whether Harbor is logged in
- Whether kubectl / kubeconfig is available
- Different network and certificate environments

---

## Key Points of This Section

1. The essential role of Runner / Agent
2. The difference between definition layer and execution layer
3. Common toolchains required on an executor
4. How to simulate an executor with a local script
5. Why CI/CD configurations can't run without an executor

## One-Sentence Summary

CI/CD configuration files define "how to write the process", while GitLab Runner and Jenkins Agent solve "who actually runs this process".