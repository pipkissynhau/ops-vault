# 21-CI-CD Troubleshooting Case: Analysis of Build Failure, Push Failure, and Deployment Failure

## Document Notes

This is the 21st note in the 08-CI-CD learning path.

The previous 01-20 sections have already established a minimal deployment pipeline and gradually completed:

- Manual deployment
- Image building and tagging
- Harbor repository
- GitLab CI / Jenkins Pipeline
- Deployment rolling update
- Helm
- Multi-environment deployment
- Executor
- Security basics
- Harbor advanced
- GitOps basics
- Blue/green / Canary / Gray deployment basics

In this article, the focus is no longer on "how to deploy", but rather:

**When deployment fails, how to troubleshoot by segmenting the pipeline.**

This article does not discuss generic advice like "check logs more often", but instead directly focuses on the minimal pipeline you've already implemented, breaking common failures into three categories:

1. Build failure
2. Push failure
3. Deployment failure

And writes it in a way that can be directly reproduced and observed in your current environment.

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Existing minimal application `manual-web`

## Tags

#Kubernetes #CI-CD #TheBarrier. #Docker #Harbor #Deployment #Helm #rollout #ImagePullBackOff #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should be able to:

1. Understand why troubleshooting must be segmented by pipeline
2. Know where to look for build failure, push failure, and deployment failure
3. Reproduce several common failure types in the current environment
4. Quickly determine where the problem likely occurs based on symptoms
5. Explain the troubleshooting order for a minimal deployment pipeline

## This Article's Troubleshooting Mainline

At this stage, fix the problem into 3 major segments:

### Segment 1: Build Failure
Occurs at:

- Local build
- Runner / Agent build
- Dockerfile processing stage

### Segment 2: Push Failure
Occurs at:

- Harbor login
- Tag error
- Repository permissions
- Repository connection

### Segment 3: Deployment Failure
Occurs at:

- Deployment / Helm update
- Pod pulling image
- Container startup
- Rollout convergence
- Business validation

Once these three segments are fixed, many problems will no longer be chaotic.

---

## Part 1: Establish a Fixed "Segmented Troubleshooting" Approach

The current minimal deployment pipeline is:

Application content  
→ Image building  
→ Image storage  
→ Cluster configuration  
→ Cluster deployment  
→ Deployment validation

In the future, when troubleshooting, do not immediately say:

- "CI/CD failed"
- "K8s has issues"
- "Harbor cannot pull"

Instead, first determine:

### 1) Is the build stage failed?

For example:

- Dockerfile error
- Build context error
- Local image cannot start

### 2) Is the image failed to enter Harbor?

For example:

- Push failure
- Tag error
- Harbor permission insufficient

### 3) Is the image in Harbor but the cluster cannot use it properly?

For example:

- Image written incorrectly
- Pod cannot pull image
- Rollout stuck
- Page still shows old content

### Understanding at This Stage

The first step in troubleshooting is not to look at commands, but to segment the problem first.

---

## Part 2: Build Failure - Focus on the Build Stage First, Don't Rush to Suspect Harbor and K8s

Issues in this segment usually occur at:

- `docker build`
- Local image verification
- Dockerfile
- Build context

### Most Common Symptoms

- `docker build` directly reports an error
- Build succeeds but local content is incorrect
- Build succeeds but container cannot start

---

## Part 3: Troubleshooting Case 1 - Dockerfile Reference File Error

### Fault Target

Simulate the most common build failure:

**The Dockerfile COPYs a non-existent file.**

### Step 1: Enter the Experiment Directory

    cd ~/08-ci-cd/01-manual-release

### Step 2: Intentionally Modify Dockerfile

Change the original:

    COPY index.html /usr/share/nginx/html/index.html

To the error version:

    cat > Dockerfile <<'EOF'
    FROM nginx:1.27
    COPY index2.html /usr/share/nginx/html/index.html
    EXPOSE 80
    EOF

### Step 3: Execute Build

    docker build -t manual-web:build-fail-1 .

### Expected Symptoms

Most likely to report:

- `COPY failed`
- Cannot find `index2.html`
- Build failure

### Troubleshooting Conclusion

This issue belongs to:

**Build stage failure**

It hasn't reached Harbor yet, nor has it reached K8s.

### Step 4: Restore Dockerfile

    cat > Dockerfile <<'EOF'
    FROM nginx:1.27
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80
    EOF

### Understanding at This Stage

As long as `docker build` hasn't passed, don't suspect:

- Harbor
- Deployment
- rollout

The problem lies in the build stage.

---

## Part 4: Troubleshooting Case 2 - Build Succeeds, But Local Content Is Incorrect

This type of issue is very common and can easily mislead subsequent troubleshooting.

### Fault Target

Simulate:

- Build can succeed
- But the image content is not what you want

### Step 1: Modify Page Content

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual wrong content</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: wrong-build-content</p>
      </body>
    </html>
    EOF

### Step 2: Build Image

    docker build -t manual-web:check-local .

### Step 3: Local Run Verification

docker run -d --name manual-web-check -p 18080:80 manual-web:check-local
curl http://127.0.0.1:18080
docker rm -f manual-web-check

### Phenomenon

You will find:

- build has no issues
- container can start
- but content is incorrect

### Troubleshooting Conclusion

This issue still belongs to:

**Build phase problem**

Not because K8s hasn't been updated, but because the content you built in is inherently wrong.

### Current understanding to establish at this step

In the future, when page results are incorrect, do not immediately suspect rollout.  
First ask yourself:

> Is the content of the image built locally actually correct?

---

## Part Five: Push Failure - Image Not Yet in Harbor, Do Not Check Pod

This issue occurs in:

- `docker login`
- `docker tag`
- `docker push`

### Most Common Phenomenon

- push is rejected
- tag written incorrectly
- Harbor address written incorrectly
- login successful but cannot push to target project
- tag not visible on the page

---

## Part Six: Troubleshooting Case 3 - Harbor Address Written Incorrectly Causes Push Failure

### Fault Objective

Simulate:

- build successful
- but push address written incorrectly

### Step 1: Prepare an image that can be pushed

    cd ~/08-ci-cd/01-manual-release
    docker build -t manual-web:push-fail-1 .

### Step 2: Intentionally tag with an incorrect Harbor address

    docker tag manual-web:push-fail-1 wrong-harbor.example.com/test/manual-web:push-fail-1

### Step 3: Execute push

    docker push wrong-harbor.example.com/test/manual-web:push-fail-1

### Expected Phenomenon

You will likely see:

- domain name cannot be resolved
- connection failed
- TLS or connection error

### Troubleshooting Conclusion

This issue belongs to:

**Image repository phase failure**

The image hasn't entered K8s yet.

### Current understanding to establish at this step

If `docker push` hasn't succeeded, do not rush to check:

- Deployment
- Pod
- rollout

Because the cluster has no opportunity to use this image yet.

---

## Part Seven: Troubleshooting Case 4 - Harbor Has No Such Tag

This issue is more subtle than push command error.

### Scenario

You think you pushed:

    manual-web:test-21

But actually, there is no such tag in Harbor page.

### Troubleshooting Steps

#### Step 1: Confirm on Harbor page

Enter:

- Project: `test`
- Repository: `manual-web`

Confirm:

- Whether `test-21` actually exists

#### Step 2: Confirm tag is correct locally

    docker images | grep manual-web

#### Step 3: Confirm push command was actually executed

For example, have you executed:

    docker push your Harbor domain/test/manual-web:test-21

### Troubleshooting Conclusion

This issue also belongs to:

**Image repository phase failure or incomplete**

### Current understanding to establish at this step

In the future, when seeing Pod cannot pull image, do not only check K8s,  
Always manually confirm on Harbor page first:

> Is this tag actually present?

---

## Part Eight: Release Failure - Image Entered Harbor Does Not Mean Environment Can Use It Normally

This is the most complex and common scenario.

Release failure usually occurs in:

- Deployment image written incorrectly
- Pod cannot pull image
- Container startup failure
- rollout stuck
- page still shows old content

---

## Part Nine: Troubleshooting Case 5 - Pod Cannot Pull Image

This is the most typical and common type of failure.

### Fault Objective

Intentionally make Deployment use a non-existent tag.

### Step 1: Execute erroneous release

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:not-exist-21

### Step 2: Check Pod

    kubectl -n test get pods

### Expected Phenomenon

You will likely see:

- `ErrImagePull`
- `ImagePullBackOff`

### Step 3: Check Pod details

    kubectl -n test describe pod Pod name

Focus on `Events`.

### Expected Phenomenon

Usually you will see similar:

- Image pull failed
- Tag does not exist in repository
- Permission or connection error

### Troubleshooting Conclusion

This issue belongs to:

**Image pull failure during release phase**

It is neither a build issue nor a page content issue.

### Current understanding to establish at this step

When seeing `ImagePullBackOff`, the first reaction should be:

1. Does this tag exist in Harbor page
2. Is Deployment image written correctly
3. Is Harbor authentication/certificates/connection normal

---

## Part Ten: Troubleshooting Case 6 - rollout Stuck

### Fault Objective

Observe what happens when Pod cannot be ready.

### Step 1: Still use the above incorrect image or other configuration causing Pod failure

Execute:

    kubectl -n test rollout status deployment/manual-web

### Phenomenon

It will not succeed quickly, but wait, get stuck, or eventually timeout.

### What to do after Step 1

Execute in order:

    kubectl -n test get deploy
    kubectl -n test get rs
    kubectl -n test get pods
    kubectl -n test describe pod Pod name

### Troubleshooting Conclusion

rollout stuck is just a phenomenon, the real cause is usually in:

- Pod not started
- Pod not Ready
- Pod cannot pull image
- Container startup failure

### Current understanding to establish at this step

In the future, when seeing rollout stuck, do not only focus on `rollout status` itself.  
It is just a "controller hasn't converged" indication, the real cause should be found at Pod level.

1. Deployment didn't actually update to the new tag  
2. Helm values weren't changed successfully  
3. Service is accessing the wrong set of Pods  
4. The content you built/pushed was incorrect from the start  
5. You used the same tag to overwrite, causing version confusion  

### Troubleshooting Order  

#### Step 1: Confirm Deployment's actual image  

    kubectl -n test describe deploy manual-web | grep -A3 Image  

#### Step 2: If using Helm, confirm values  

    cd ~/08-ci-cd/08-helm-lab/manual-web  
    cat values-test.yaml  

#### Step 3: Confirm the tag exists in Harbor  

Check on the Harbor page.  

#### Step 4: Confirm the page access result  

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh  

After entering, execute:  

    wget -qO- http://manual-web.test.svc.cluster.local  

#### Step 5: Verify locally if needed  

    docker run -d --name local-check -p 18080:80 manual-web:your-local-tag  
    curl http://127.0.0.1:18080  
    docker rm -f local-check  

### Troubleshooting Conclusion  

An old page doesn't necessarily mean the issue is in K8s.  
It's likely the problem occurred earlier in the chain.  

### Understanding to Establish at This Step  

"The page didn't change" is the final phenomenon,  
Real troubleshooting must trace back through:  

- Content  
- Build  
- Push  
- Image reference  
- Pod  
- Service  

Step by step.  

---

## Part 12: Troubleshooting Case 8 - Helm upgrade results are incorrect  

This type of issue is very common after Helm advancement.  

### Common Causes  

1. Modified the wrong values file  
2. Changed values but used incorrect field names, failing to override properly  
3. `helm template` rendering result differs from expectation  
4. Mixed up release name or namespace  
5. Harbor tag doesn't exist  

### Troubleshooting Steps  

#### Step 1: Confirm which values file you modified  

    cd ~/08-ci-cd/08-helm-lab/manual-web  
    cat values-test.yaml  

#### Step 2: Check the rendering result  

    helm template manual-web . -f values-test.yaml -n test  

Focus on:  

- image  
- replicas  
- namespace  

#### Step 3: Check release history  

    helm history manual-web-test -n test  

#### Step 4: Check Deployment's actual status  

    kubectl -n test describe deploy manual-web | grep -A3 Image  

### Troubleshooting Conclusion  

Helm issues often aren't "Helm is broken," but rather:  

- Configuration override didn't take effect  
- Rendering result differs from expectation  
- Operated on the wrong release  

### Understanding to Establish at This Step  

Helm troubleshooting must rely heavily on:  

- `cat values-xxx.yaml`  
- `helm template`  
- `helm history`  

Rather than just checking if `helm upgrade` succeeded.  

---

## Part 13: Categorizing Common Failures by Phenomenon  

This section is crucial; you can use it directly as a quick reference.  

### Phenomenon 1: `docker build` directly reports an error  

Likely belongs to:  

- Build phase issues  

Check first:  

- Dockerfile  
- File path  
- Build context  

### Phenomenon 2: `docker push` fails  

Likely belongs to:  

- Image repository phase issues  

Check first:  

- Harbor address  
- Login status  
- tag  
- Permissions  

### Phenomenon 3: `ImagePullBackOff`  

Likely belongs to:  

- Image pull issues during deployment  

Check first:  

- Does Harbor have this tag?  
- Is the image written correctly?  
- Harbor authentication/trust/connection  

### Phenomenon 4: Rollout is stuck  

Likely belongs to:  

- Pod not ready  
- Image pull failure  
- Container startup failure  

Check first:  

- `kubectl get pods`  
- `kubectl describe pod`  
- `kubectl logs`  

### Phenomenon 5: Page still shows old content  

Likely belongs to:  

- Image didn't change  
- Values didn't take effect  
- Tag confusion  
- Service accessing the wrong target  
- Local build content was incorrect from the start  

---

## Part 14: Recommended Fixed Troubleshooting Order at This Stage  

From now on, you can follow this order consistently.  

### Step 1: Confirm content  

- `cat index.html`  
- What exactly did you change?  

### Step 2: Confirm build  

- `docker build`  
- Local run  
- Does the page content match expectations?  

### Step 3: Confirm repository entry  

- `docker push`  
- Confirm tag on Harbor page  

### Step 4: Confirm configuration/deployment  

- Deployment image  
- Helm values  
- Helm template  

### Step 5: Confirm Pod operation  

- `kubectl get pods`  
- `kubectl describe pod`  
- `kubectl logs`  

### Step 6: Confirm Service/page result  

- Cluster internal wget/curl  
- Page version content  

### Understanding to Establish at This Step  

Troubleshooting isn't about "inspiration,"  
It's about following a stable order to eliminate segments of the chain.  

---

## Part 15: This Section's Practice Exercise  

### Exercise 1: Reproduce a build failure  

Requirements:  

- Intentionally modify Dockerfile incorrectly  
- Observe build error  
- Restore and rebuild successfully  

### Exercise 2: Reproduce a push failure  

Requirements:  

- Intentionally write wrong Harbor address  
- Observe push error  
- Explain which segment the issue belongs to  

### Exercise 3: Reproduce a deployment failure  

Requirements:  

- Intentionally point Deployment to a non-existent tag  
- Observe `ImagePullBackOff`  
- Check `describe pod`  
- Explain which segment the issue belongs to  

### Exercise 4: Answer the following 5 questions yourself  

1. Why troubleshoot by segmenting the chain?  
2. Where to check first when build fails?  
3. Where to check first when push fails?  
4. What to check first for `ImagePullBackOff`?  
5. Why can't you blame rollout alone when the page doesn't change?  

If you can explain these 5 questions yourself, you've mastered this section.  

---

## Content You Should Be Able to Explain After This Section  

After completing this section, you should be able to clearly explain the following statement:

CI/CD troubleshooting is not about memorizing many commands, but first segmenting the problem along the pipeline.  
If `docker build` fails, the issue is in the build stage; if `docker push` fails, the issue is in the image push stage; if a Pod shows `ImagePullBackOff`, the problem is typically in the image pull phase of the deployment stage.  
When rollout gets stuck, it's often just a surface-level phenomenon — the real cause usually requires checking Pods, Events, and logs.  
Issues like "the page is still the old version" cannot be solely blamed on K8s, because any failure in content, build, push, image reference, Helm values, or Service paths can ultimately manifest as the page not changing.

## Common Issues and Troubleshooting Directions

### Issue 1: Why does "deployment failure" have so many different causes

Because deployment is just the latter half of the pipeline — errors in the build/push phase can propagate all the way to the later stages.

### Issue 2: Why do I still get confused during troubleshooting even though I know many commands

Because commands are just tools — true stability comes from "segmenting to determine where the issue is."

### Issue 3: Are Helm issues harder to troubleshoot

Not necessarily harder, but they add an extra layer compared to direct kubectl:
- values
- template rendering
- release history

So use more:
- `cat values`
- `helm template`
- `helm history`

---

## Key Takeaways from This Article

1. Basic segmentation of build failure, push failure, and deployment failure
2. Typical manifestations of several common failures
3. Priority troubleshooting directions for different manifestations
4. The most valuable troubleshooting order at the current stage
5. How to upgrade from "knowing how to deploy" to "being able to locate issues"

## One-Sentence Summary

The key to CI/CD troubleshooting is not to flip through logs everywhere when seeing a failure, but first segment the problem into "build, push, deployment, and verification" stages, then trace the issue layer by layer along the pipeline.