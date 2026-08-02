# 16-Helm Advanced: Values Overriding, Environment Splitting, and Template Reuse

## Document Notes

This is the 16th note in the 08-CI-CD learning path.

The 8th note already completed Helm's minimal introduction:

- What is a Chart
- What are templates
- What are values
- How to perform helm install / upgrade / rollback

The 13th note then began establishing multi-environment deployment thinking:

- dev
- test
- prod
- How to split differences in namespace, image tag, replicas

This note connects these two lines of thought, focusing on solving this key issue:

**When Helm truly begins to handle multi-environment configurations, how should values be split, overridden, and reused.**

This note moves beyond the "a values.yaml can run" level, entering the practical techniques you'll repeatedly use later:

- How to place default values
- How to split dev / test / prod
- What should be written in values
- What should remain in templates
- How to reuse the same Chart across multiple environments
- Why disorganized values can actually make maintenance harder

This article continues to align with the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace
- Existing minimal Helm Chart: `manual-web`

## Tags

#Kubernetes #CI-CD #Helm #Values #Multi-environmentRelease #TemplateReuse #Chart #HelmStep #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understand why Helm values need layering and overriding
2. Understand what belongs in default values vs environment-specific values
3. Understand how to split multi-environment values files
4. Understand what belongs in template structure vs values parameters
5. Create dev / test value sets for the current Helm Chart
6. Execute an install / upgrade with value overriding
7. Explain the basicThinking. of "Helm template reuse"

## This Article's Experimental Flow

This article is divided into 4 sections:

1. First organize the current Chart's default values and environment differences
2. Split out `values-dev.yaml` and `values-test.yaml`
3. Render and publish using `-f` overriding methods
4. Summarize the minimal reuse rules for Helm in multi-environment deployments

---

## Part 1: First Review Your Current Helm Chart Status

Enter the Helm Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

Check current files:

    ls
    ls templates
    cat values.yaml

If you've completed the 8th and 13th notes earlier, you should have at least viewed these contents:

- `Chart.yaml`
- `values.yaml`
- `templates/deployment.yaml`
- `templates/service.yaml`
- Possibly also `values-dev.yaml`
- Possibly also `values-test.yaml`

### Understanding to Establish at This Step

By now, your Helm Chart has moved beyond "being able to run" and is starting to carry "multi-environment differences."

So the focus of this article isn't creating a new Chart, but rather:

**Establishing a clearer values structure on an existing Chart.**

---

## Part 2: Why Values Need Layering, Not All in One File

This is a critical question.

If you have only one environment, a single `values.yaml` might suffice.  
But once you start encountering:

- dev
- test
- prod

You'll immediately face two problems:

### Problem 1: Putting all environment parameters into one values.yaml becomes messy

For example:

- image tag
- replicaCount
- namespace
- Service configuration
- Ingress domain
- Resource limits
- HPA parameters

After mixing all in one file, you'll struggle to determine:

- Which are default structures
- Which are dev differences
- Which are test differences
- Which are prod-specific overrides

### Problem 2: Copying a full values file for each environment leads to repetition

If you completely copy a large values file for each environment, you'll quickly face:

- Lots of repeated content
- Changing a common parameter requires editing three files
- Prone to omissions
- Files become harder to read

### Understanding to Establish at This Step

So the most reasonable direction for Helm values is typically not:

- Putting everything in one file
- Or fully copying a values file for each environment

Rather:

**Keep default values for commonalities and use environment-specific files for overrides.**

---

## Part 3: First Distinguish "Default Commonality" and "Environment Differences"

This section is the most important foundational thinking for the entire article.

### 1) Default Commonality

So-called default commonality refers to:

- All environments are basically the same
- Rarely changes with environment differences
- MoreDiverse "application structure" or "basic conventions"

In the current experiment, this typically includes:

- Service type = ClusterIP
- Service port = 80
- containerPort = 80
- Application name conventions
- Template structure itself

### 2) Environment Differences

So-called environment differences refers to:

- Changes with environment
- MoreDiverse "deployment target" parameters

In the current experiment, the most typical are:

- namespace
- replicaCount
- image.tag
- Future may include domain, resources, HPA switches, etc

### Understanding to Establish at This Step

The core of this article is to separate these two categories:

- Place commonalities in default values
- Place differences in environment-specific override files

---

## Part 4: Organize Current values.yaml, Retain Default Commonality

This section begins the hands-on work.

### Step 1: Enter the Chart Root Directory

    cd ~/08-ci-cd/08-helm-lab/manual-web

### Step 2: Transform values.yaml into "Default Commonality Values" Version

Execute:

    cat > values.yaml <<'EOF'
    replicaCount: 1

    image:
      repository: your Harbor domain/test/manual-web
      tag: "latest"
      pullPolicy: IfNotPresent

    service:
      type: ClusterIP
      port: 80

    containerPort: 80

    namespace: test
    EOF

Note: Replace `你的Harbor域名` with your actual Harbor domain.

### Understanding This Step

Here, although still retaining: /think

- `replicaCount`
- `tag`
- `namespace`

But currently you need to understand this as:

**A set of the most conservative, most basic default values.**

Later, dev / test / prod are not copying this file, but overriding it based on this foundation.

---

## Part 5: Create values-dev.yaml, Only Write Dev Differences

### Step 1: Create the Dev Environment Override File

Execute:

    cat > values-dev.yaml <<'EOF'
    replicaCount: 1

    image:
      tag: "dev-13"

    namespace: dev
    EOF

### Current Understanding to Establish Here

This file intentionally only writes the most critical differences:

- Replica count
- Image tag
- Namespace

It does not repeat writing:

- service.type
- service.port
- containerPort
- pullPolicy
- repository

Because these values have already been defined in the default `values.yaml`.

In other words:

**values-dev.yaml is only responsible for overriding, not redefining everything.**

---

## Part 6: Create values-test.yaml, Only Write Test Differences

### Step 1: Create the Test Environment Override File

Execute:

    cat > values-test.yaml <<'EOF'
    replicaCount: 2

    image:
      tag: "test-13"

    namespace: test
    EOF

### Current Understanding to Establish Here

Here you should clearly see:

- dev and test share the same template
- Share most default values
- Differ only at the most critical few points

This is the core idea of Helm's multi-environment values.

---

## Part 7: First Use helm template to See Dev / Test Override Effects

This part first checks the rendered results without installing/upgrading.

### Render Dev

Execute:

    helm template manual-web . -f values-dev.yaml -n dev

### Render Test

Execute:

    helm template manual-web . -f values-test.yaml -n test

### What to Focus On

Mainly check these fields:

1. Deployment's namespace
2. replicas
3. image.tag
4. Service's namespace

### Current Understanding to Establish Here

Through this step you should clearly see:

- The template remains unchanged
- values.yaml is still the same base default values
- The rendered results changed only because of `-f values-dev.yaml` or `-f values-test.yaml`

This is Helm's "template reuse + parameter override."

---

## Part 8: Why `-f values-dev.yaml` is "Override," Not "Replace"

This point is easy to confuse.

Execute:

    helm template manual-web . -f values-dev.yaml

This does not mean Helm only looks at `values-dev.yaml`.  
Rather:

1. First read the default `values.yaml`
2. Then override the default values with the same-named fields in `values-dev.yaml`

In other words:

- Default values remain effective
- Environment files only override differences

### Current Understanding to Establish Here

You can remember it like this:

- `values.yaml`: Base foundation
- `values-dev.yaml`: Modify dev differences on the foundation
- `values-test.yaml`: Modify test differences on the foundation

---

## Part 9: Hands-on - Install dev Environment Release Using values-dev.yaml

This part begins the actual deployment.

### Step 1: Install dev Environment Release

Execute:

    helm install manual-web-dev . -f values-dev.yaml -n dev --create-namespace

### Step 2: Check Release List

Execute:

    helm list -A

### Step 3: Check dev Environment Resources

Execute:

    kubectl -n dev get deploy,pods,svc

### Step 4: Verify Page Content

Execute:

    kubectl -n dev run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.dev.svc.cluster.local

### Expected Outcome

Should see:

    version: dev-13

### Current Understanding to Establish Here

At this point you should clearly understand:

- Same Chart
- Use dev's values to override
- Install as a dev environment release

This is the actual form of Helm managing multi-environments.

---

## Part 10: Hands-on - Install test Environment Release Using values-test.yaml

### Step 1: Install test Environment Release

Execute:

    helm install manual-web-test . -f values-test.yaml -n test --create-namespace

### Step 2: Check Release List

Execute:

    helm list -A

### Step 3: Check test Environment Resources

Execute:

    kubectl -n test get deploy,pods,svc

### Step 4: Verify Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Outcome

Should see:

    version: test-13

### Current Understanding to Establish Here

Now you have created two releases:

- `manual-web-dev`
- `manual-web-test`

They:

- Come from the same Chart
- Form different environment instances due to different values

This is Helm's true advantage in multi-environment scenarios.

---

## Part 11: Now Look Back to Understand What "Template Reuse" Actually Reuses

By this point, explaining "template reuse" is no longer abstract.

### What is Reused is Not the Environment Results

The final rendered YAML for dev and test are different.

### What is Reused is the Template Structure

That is:

- How a Deployment looks
- How a Service looks
- How labels / selector / ports are organized

These structures do not need to be written out for each environment.

### Understanding to Establish at This Step

The essence of template reuse is:

**Share a single resource structure, and let environment differences be reflected through parameters.**

---

## Part 12: What Should Go Into values and What Shouldn't Be Randomly Placed

This section is very important.

Not everything is suitable for being piled into values.

### More Suitable for values

- image.repository
- image.tag
- replicaCount
- namespace
- service.port
- ingress.host
- resources
- feature switch parameters

### Not Suitable to Put Everything into values at the Start

- Already fixed template structures
- Parameterizing all minor details
- Unused future extension points

### Understanding to Establish at This Step

The goal of values is not "parameterizing everything", but:

**Extract parameters that will actually change and are worth splitting by environment.**

Over-parameterization can actually make the Chart harder to read.

---

## Part 13: Hands-on - Helm upgrade with values Coverage

This section lets you experience the upgrade method of "only changing values in the same environment".

### Step 1: Prepare a new dev image tag

Assume Harbor already has:

    dev-16

If not, you can rebuild and push from the application directory first.

### Step 2: Modify values-dev.yaml

Execute:

    cat > values-dev.yaml <<'EOF'
    replicaCount: 1

    image:
      tag: "dev-16"

    namespace: dev
    EOF

### Step 3: Execute upgrade

Execute:

    helm upgrade manual-web-dev . -f values-dev.yaml -n dev

### Step 4: Check release history

Execute:

    helm history manual-web-dev -n dev

### Step 5: Verify page content

Execute:

    kubectl -n dev run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.dev.svc.cluster.local

### Understanding to Establish at This Step

Here you didn't change the template,  
you only changed:

- `values-dev.yaml`

Then Helm helps you:

- Re-render
- Update the release
- Trigger K8s updates

This is the most direct value of values coverage.

---

## Part 14: Naming of Multi-environment values Files, What's Most Suitable at This Stage

At this stage, keep the simplest naming:

- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`
- `values-prod.yaml`

### Why This Is Sufficient

Because your current focus is:

- First learn the environment splitting approach
- First learn values coverage
- First learn to install releases by environment

Rather than designing a complex directory tree from the start.

### Understanding to Establish at This Step

At this stage:

**Clear files are more important than fancy ones.**

---

## Part 15: Recommended Helm Multi-environment Rules at This Stage

It's recommended to fix these rules first.

### Rule 1: values.yaml for base default values

### Rule 2: Environment values only write differences, don't fully copy

### Rule 3: namespace, image.tag, replicaCount are the three most valuable items to split first

### Rule 4: One release name per environment, avoid confusion

For example:

- `manual-web-dev`
- `manual-web-test`

### Rule 5: First make template reuse solid, then gradually add parameter items

Don't parameterize all fields from the start.

---

## Part 16: Intentionally Make an "Over-copied values" Anti-Example

This section is recommended to feel it yourself.

### Step 1: Copy a values-bad-test.yaml

Execute:

    cp values.yaml values-bad-test.yaml

Then manually change all fields to test environment values.

### Step 2: Compare `values-bad-test.yaml` and `values-test.yaml`

Execute:

    cat values-bad-test.yaml
    cat values-test.yaml

### Understanding to Establish at This Step

You'll find:

- Full replication seems convenient at first
- But there's too much repetition
- Once default values change, it's easy to miss synchronization

This is why multi-environment values are more recommended:

**Default values + difference coverage**

---

## Part 17: This Section's Mini Exercise

### Exercise 1: First Implement the prod Approach, Not Necessarily Deploy Immediately

Create:

    values-prod.yaml

At least write:

- replicaCount
- image.tag
- namespace

Even if you don't install it now, form the parameter thinking first.

### Exercise 2: Do a dev Environment Upgrade Again

Requirements:

- Prepare a new dev tag
- Modify `values-dev.yaml`
- Execute `helm upgrade`
- Verify the page result

### Exercise 3: Answer the Following 5 Questions Yourself

1. Why do values need layering?
2. What's suitable for default values and environment values?
3. Why are environment values not recommended for full replication?
4. What does Helm template reuse actually reuse?
5. What are the most valuable environment difference items to split first at this stage?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content You Should Be Able to Explain After This Section

After completing this section, it's recommended to be able to explain the following:

Helm values need layering because the same application typically has commonalities and differences across environments.  
`values.yaml` is more suitable for base default values, while `values-dev.yaml`, `values-test.yaml`, and other environment files are better suited for only covering the truly changing parts, such as namespace, image.tag, and replicaCount.  
The benefit of this approach is that the same Chart template can be reused by multiple environments without replicating nearly identical YAMLs.  
Helm's template reuse essentially reuses the resource structure; the actual differences between environments are parameters, not the entire template logic.

## Common Issues and Troubleshooting Directions

### Issue 1: Why is the rendered result different from what I expect?

Prioritize checking:

- Default values in `values.yaml`
- Whether the fields in the environment override file are correctly written
- What the actual rendered result of `helm template` is

### Issue 2: Why are dev and test, which are the same Chart, different?

Because the Chart reuses the template, and the results are determined by different values.

### Question 3: Should all fields be made into values?

No.  
Only fields that "change and are worth managing by environment" should be extracted into values.

---

## Key Takeaways of This Article

1. Layering and Overriding Logic for values  
2. Responsibilities division between default values and environment-specific values  
3. Helm's ability to reuse templates across multiple environments  
4. Current most suitable parameters to extract for environments  
5. How to use `-f values-xxx.yaml` for install / upgrade

## One-Sentence Summary

The core of Helm's multi-environment management is not replicating a full configuration for each environment, but allowing the same Chart to share template structure, and managing actual environment differences through default values and environment-overriding values.