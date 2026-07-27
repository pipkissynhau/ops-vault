# 09-CI-CD Practical Release: From Building Images to Deploying in Kubernetes

## Document Description

This article is the 9th note in the 08-CI-CD learning pathway.

Previously, articles 01 to 08 covered the following topics separately:

1. The minimal manual release process
2. Segmented understanding of the release process
3. Image building and production-style tags
4. Projects, repositories, tags, and Robot accounts in Harbor
5. Basic understanding of GitLab CI pipelines
6. Basic understanding of Jenkins Pipeline
7. Deployment, ReplicaSet, Pod, and rolling update mechanisms
8. Basic template-based release and rollback using Helm

This article does not focus on any single topic but combines the previous content into a complete practical workflow:

Application Content  
→ Build images  
→ Apply production-style tags  
→ Push images to Harbor  
→ Deploy using kubectl or Helm in Kubernetes  
→ Monitor the rollout process  
→ Verify the results  
→ Troubleshoot issues based on the process

This article continues to use the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Private Harbor repository
- The `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Helm #Deployment #rollout #Image Building #Practical Release #Hands-on Notes

## Learning Objectives

After completing this article, you should be able to:

1. Independently execute the entire minimal release process
2. Switch between using kubectl and Helm for deployment
3. Use production-style image tags
4. Monitor the rollout process and determine if the release was successful
5. Identify which part of the process caused a failure when releases fail
6. Describe the entire release process in detail

## Experimental Design for This Article

This article offers two optional deployment paths:

### Path A: Deployment using kubectl

Suitable for understanding the most direct release mechanism.

### Path B: Deployment using Helm

Suitable for understanding template-based, parameterized, and release management processes.

Both paths use the same image build results, so you will clearly see that:

- Image building is the first step
- kubectl and Helm are just two different ways to execute the second step

## Experimental Preparation

### 1) Enter the experimental directory

Continue using the existing directories for the application content:

    cd ~/08-ci-cd/01-manual-release

Continue using the existing directory for the Helm-related files:

    cd ~/08-ci-cd/08-helm-lab/manual-web

### 2) Verify that the cluster and namespace are functioning properly

Execute the following commands:

    kubectl get nodes
    kubectl get ns
    kubectl -n test get deploy,pods,svc

### 3) Ensure that Harbor is accessible

Run:

    docker login your_Harbor_domain_name

### 4) Confirm that Helm is available

Run:

    helm version

## Part 1: Prepare a New Release Version

It is recommended to start with a brand-new version, such as `v9`.

### Step 1: Modify the application content

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Modify the `index.html` file:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v9</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v9</p>
      </body>
    </html>
    EOF

Check the content:

    cat index.html

### Understanding Required for This Step

The starting point of the release process is always a change in the application content.  
In this experiment, the change is the version number being updated from an old value to `v9`.

## Part 2: Build Images

### Step 1: First, build a local learning tag

Run:

    docker build -t manual-web:v9 .

View the images:

    docker images | grep manual-web

### Step 2: Run locally for verification

Execute the following commands:

    docker run -d --name manual-web-v9 -p 18080:80 manual-web:v9
    curl http://127.0.0.1:18080
    docker rm -f manual-web-v9

### Expected Outcome

The output should include:

    version: v9

### Understanding Required for This Step

This step confirms that:

- The new content has been included in the image
- The image can run locally
- No issues have occurred so far with Harbor or Kubernetes

## Part 3: Apply Production-style Tags to the Image

Instead of just keeping the learning tag🔤 Translate the following text from Chinese_simplified to English. Preserve the original formatting, sentence structure, and terminology. Ensure that every word and sentence is translated as closely as possible to the original meaning, without summarizing or omitting any part of the content. The translation must be faithful, detailed, and maintain the original length and complexity. Deliver the translation exactly as required, without any additional commentary or explanation, and ensure the 🔤 symbols are removed in the final output.