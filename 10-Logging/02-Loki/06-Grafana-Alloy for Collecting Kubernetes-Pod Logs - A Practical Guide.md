# 06-Grafana-Alloy for Collecting Kubernetes-Pod Logs: A Practical Guide

## Document Overview

This article is the sixth in a series dedicated to learning about Loki. It aims to guide you through deploying Grafana Alloy in a Kubernetes cluster and setting up log collection from Kubernetes Pods, which are then forwarded to Loki.

Previous articles have covered:

    01-Understanding Loki Basics and Setting Up the Experimental Environment
    02-Practical Observations of Loki’s Architecture and Component Responsibilities
    03-Comparing Different Loki Deployment Models and Selecting an Approach for Experiments
    04-Practical Guide to Deploying Loki in a Standalone Mode Using Helm
    05-How to Connect Loki’s Backend Storage to MinIO

In Article 04, it was confirmed that the Loki server can manually write and query logs.

Article 05 demonstrated how to connect Loki’s backend storage to MinIO.

This article begins to establish a real-world log collection pipeline for Kubernetes Pods:

    Kubernetes Pod stdout / stderr
      ↓
    Kubernetes API / Node Container Logs
      ↓
    Grafana Alloy
      ↓
    Loki Gateway
      ↓
    Loki
      ↓
    LogQL Queries
      ↓
    Grafana Explore / Dashboard

This article addresses the following key questions:

- What is Alloy?
- How does Alloy differ from Promtail, Fluent Bit, and Filebeat?
- Why should Alloy be preferred in new environments?
- Why is Alloy usually deployed as a DaemonSet in Kubernetes?
- How to deploy Alloy using Helm?
- How to configure Alloy to detect Pods?
- How to add namespace, pod, container, node, app, and cluster labels to logs?
- How to write Pod logs to Loki?
- How to verify that Alloy is running properly?
- How to confirm that Loki is receiving Pod logs?
- How to use LogQL to query logs for apps like app-demo or nginx-demo?
- What are the common troubleshooting steps when Alloy fails to collect logs?
- What considerations should be taken when using Alloy to collect Pod logs in a production environment?

---

## Tags

#Loki #GrafanaAlloy #Kubernetes #PodLogs #LogCollection #DaemonSet #LogQL #Grafana #SRE #Observability #LogTroubleshooting

---

## Recommended Reading Path

Recommended reading path:

    10-Logs/02-Loki/06-Grafana-Alloy for Collecting Kubernetes-Pod Logs.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the role of Alloy in the Loki log system.
    2. Deploy Grafana Alloy using Helm.
    3. Set up Alloy as a DaemonSet on Kubernetes nodes.
    4. Configure Alloy to detect Kubernetes Pods.
    5. Configure Alloy to add namespace/pod/container/node/app labels to logs.
    6. Set up Alloy to forward Pod logs to Loki.
    7. Verify that Alloy pods are deployed across all nodes.
    8. Ensure that no log transmissions fail with Alloy.
    9. Query logs in the app-demo Namespace within Loki.
    10. Search for logs by pod/container/app in Loki.
    11. Generate logs through a test application using Nginx.
    12. Master common troubleshooting methods when Alloy fails to collect logs.
    13. Understand the permission, labeling, performance, and security considerations related to Alloy log collection in production environments.

---

## II. Experimental Environment

### 2.1 Kubernetes Cluster

Experimental nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Container runtime:

    containerd

Namespaces:

    logging
    monitoring
    app-demo
    minio

This article will primarily use:

    logging:
        To deploy Loki and Alloy.

    app-demo:
        To deploy a test application that generates logs.

### 2.2 Pre-requisites Completed

The following prerequisites have been fulfilled:

    [ ] Loki has been deployed.
    [ ] The Loki Gateway is accessible.
    [ ] The /ready endpoint returns "ready".
    [ ] The /metrics endpoint is accessible.
    [ ] Manual log pushing/query operations were successful.
    [ ] MinIO has been integrated, or at least the Loki server is functional.
    [ ] The app-demo Namespace has been created.

Verify Loki status:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Set up port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.This is also quite common in production scenarios.

In this document, we will first use the Kubernetes API approach following the official example framework. For more advanced topics, additional options such as:

    local.file_match
    loki.source.file
    /var/log/containers
    /var/log/pods

can be explored later on.

---

## VI. Overall Architecture

After deployment, the system will have the following components connected:

    app-demo/nginx-demo Pod
      ↓
    stdout / stderr
      ↓
    Kubernetes Pod Logs
      ↓
    Alloy DaemonSet
      ↓
    loki.write
      ↓
    http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push
      ↓
    Loki
      ↓
    LogQL Queries

Architecture diagram:

    +-----------------------------+
    | app-demo Namespace          |
    |                             |
    | nginx-demo Pod              |
    | stdout / stderr             |
    +-------------+---------------+
                  |
                  v
    +-----------------------------+
    | Grafana Alloy DaemonSet     |
    | discovery.kubernetes        |
    | discovery.relabel           |
    | loki.source.kubernetes      |
    | loki.process                |
    | loki.write                  |
    +-------------+---------------+
                  |
                  v
    +-----------------------------+
    | Loki Gateway                |
    | /loki/api/v1/push           |
    +-------------+---------------+
                  |
                  v
    +-----------------------------+
    | Loki                        |
    | chunk / index / MinIO       |
    +-------------+---------------+
                  |
                  v
    +-----------------------------+
    | LogQL / Grafana Explore     |
    +-----------------------------+

---

## VII. Preparing the Test Application

If the nginx-demo has already been deployed in Chapter 01, you can skip this section directly.

### 7.1 Creating the app-demo Namespace

    kubectl create namespace app-demo

If it already exists, this step can be skipped.

### 7.2 Deploying Nginx

    kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo

### 7.3 Adding Recommended Labels to the Deployment

View the deployment:

    kubectl get deploy nginx-demo -n app-demo --show-labels

Add labels to the deployment template:

    kubectl patch deployment nginx-demo -n app-demo \
      --type='merge' \
      -p '{
        "spec": {
          "template": {
            "metadata": {
              "labels": {
                "app": "nginx-demo",
                "app.kubernetes.io/name": "nginx-demo",
                "team": "sre",
                "environment": "lab"
              }
            }
          }
        }
      }'

Wait for the deployment to roll out:

    kubectl rollout status deploy/nginx-demo -n app-demo

View the pods:

    kubectl get pod -n app-demo -o wide --show-labels

### 7.4 Exposing the Service

If there is no service yet, create one:

    kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo

View the service:

    kubectl get svc -n app-demo

    kubectl get endpoints nginx-demo -n app-demo

### 7.5 Generating Logs

Start a temporary curl Pod:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute commands inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    curl http://nginx-demo.app-demo.svc.cluster.local/healthz

Exit the pod:

    exit

### 7.6 Verifying with kubectl logs First

View the pods:

    kubectl get pod -n app-demo

View the logs:

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

You should be able to see the Nginx access logs here.

If you cannot see any logs using kubectl logs, it is also unlikely that Alloy will be able to retrieve them later on.

---

## VIII. Preparing the Helm Repository

Alloy uses the Grafana Helm repository.

### 8.1 Adding the Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 8.2 Searching for Alloy Charts

    helm search repo grafana/alloy --versions | head -20

Record the chart version:

    Chart Version:
        <actual version>

It is recommended to use a specific version for installation:

    --version <ALLOY_CHART_VERSION>

DoLoki currently has `authenabled` set to `false`, so the `tenant_id` is not configured. If multi-tenancy is enabled for production Loki, the `tenant_id` or authentication configuration must be added.1. It facilitates subsequent expansion of loki.source.file.
2. It makes it easier to track down the log paths of nodes.
3. It aligns with common file collection methods used in production scenarios.

If strict permission controls are in place, varlog mounting can be disabled in the API-based collection method.```markdown
kubectl logs <nginx-pod-name> -n app-demo --tail=50

Expected output:

    GET /
    GET /notfound
    GET /api/test

### 14.3 Using Loki to query all app-demo logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=50' | jq

### 14.4 Querying nginx-demo access logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "GET"' \
      --data-urlencode 'limit=50' | jq

### 14.5 Querying 404 logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"' \
      --data-urlencode 'limit=50' | jq

### 14.6 Querying a specific container's logs

    First, view the container labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/container/values" | jq

    To query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=50' | jq

---

## Section 15: Using Grafana Explore for Queries

If Grafana has been deployed and the Loki Data Source is added, queries can be performed directly within Grafana Explore.

### 15.1 Querying app-demo

LogQL:

    {namespace="app-demo"}

### 15.2 Querying nginx-demo

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### 15.3 Querying GET requests

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "GET"

### 15.4 Querying 404 responses

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"

### 15.5 Querying non-health check events

    {namespace="app-demo"} != "/healthz"

### 15.6 Querying ERROR, timeout, and exception logs

    {namespace="app-demo"} |~ "(?i)error|timeout|exception|panic|traceback"

Note:

    The Nginx example may not include ERROR logs.
    In Chapter 09, a JSON log demo application will be deployed for advanced LogQL practice.

---

## Section 16: Methods for Updating Alloy Configurations

### 16.1 Modifying values

Edit the file:

    values-alloy-loki.yaml

After modifications, render it first:

    helm template alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml \
      > alloy-rendered.yaml

### 16.2 Performing the upgrade

    helm upgrade alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

### 16.3 Checking the rolling update progress

    kubectl rollout status ds/alloy -n logging

If the DaemonSet name is different, use:

    kubectl get ds -n logging

Then perform the corresponding command for the correct name.

### 16.4 Verifying if new configurations have taken effect

    kubectl get cm -n logging | grep alloy

    kubectl describe pod <alloy-pod-name> -n logging

    kubectl logs <alloy-pod-name> -n logging --tail=200

### 16.5 Backing up configurations

    helm get values alloy -n logging -a > backup-values-alloy.yaml

Production recommendations:

    Include values-alloy-loki.yaml in Git.
    Perform diff comparisons before and after changes.
    Avoid directly editing ConfigMaps using kubectl.

---

## Section 17: Controlling the Scope of Log Collection

### 17.1 Risks of Collecting All Namespaces by Default

If all Pod logs are collected, this may include:

    kube-system
   4. Has the Alloy configuration been successfully loaded?
5. Is the loki.write URL correct?
6. Is the Loki Gateway accessible?
7. Is Loki /ready available?
8. Are there any errors in sending batches in Alloy logs?
9. Have there been any rejections when writing to Loki logs?
10. Were any LogQL tags incorrectly written?
11. Is the query time range correct?

Commands:

    kubectl get pod -n app-demo -o wide

    kubectl get pod -n logging -o wide | grep alloy

    kubectl logs <alloy-pod-name> -n logging --tail=300

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

### 19.2 Fault Phenomenon Two: Alloy is not running on a certain node

Check:

    kubectl get ds alloy -n logging

    kubectl get pod -n logging -o wide | grep alloy

    kubectl describe ds alloy -n logging

Common Reasons:

    The node has a taint.
    Alloy does not have any tolerations.
    There are restrictions due to the nodeSelector.
    Insufficient resources.
    Failed to pull the image.
    Restrictions from PodSecurity or security policies.

Actions:

    Add tolerations for Alloy.
    Check the nodeSelector.
    Check the Events.
    Verify access to the image repository.

### 19.3 Fault Phenomenon Three: Insufficient permissions for Alloy

Logs may show:

    forbidden
    cannot list resource pods
    cannot get resource pods/log

Check:

    kubectl auth can-i list pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i get pods/log \
      --as=system:serviceaccount:logging:alloy

Actions:

    Ensure rbac.create=true is set.
    Verify the ServiceAccount name is correct.
    Confirm that the ClusterRole / ClusterRoleBinding have been successfully created.

### 19.4 Fault Phenomenon Four: Alloy fails to connect to Loki

Alloy logs may show:

    connection refused
    no such host
    context deadline exceeded
    failed to send batch

Check the Loki Gateway:

    kubectl get svc -n logging

    kubectl get endpoints loki-gateway -n logging

Test from the logging Namespace:

    kubectl run curl-loki-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

Exit:

    exit

### 19.5 Fault Phenomenon Five: LogQL tags cannot be found

First, check the labels:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Then, check the namespace:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

Next, check the pod:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/pod/values | jq

If there are no pods / containers:

    The discovery.relabel rule may not be effective.
    The Alloy configuration may not have been updated.
    The query time range might be incorrect.
    No new logs have been written to Loki yet.

### 19.6 Fault Phenomenon Six: Unknown escape sequence

If the regex in the Alloy configuration is incorrectly written, the following error may occur:

    unknown escape sequence

Reason:

    Characters such as \S and \d need to be escaped when used within double quotes in a regex.
    It is recommended to use raw strings with single quotes.

Correct example:

    regex = `^(\S+):\/\/.+$`

Incorrect example:

    regex = "^(\S+):\/\/.+$"

---

## Twenty, Loki Side Troubleshooting

### 20.1 Check if Loki is ready

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

### 20.2 View Loki logs

    kubectl logs <loki-pod-name> -n logging --tail=300

Filtering:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|push|ingest|stream|limit|429|500"

### 20.3 Common Reasons for Loki Re### 22.3 Upgrade Alloy

    helm upgrade alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

### 22.4 Check DaemonSet Updates

    kubectl rollout status ds/alloy -n logging

If the name is different:

    kubectl get ds -n logging

### 22.5 Rollback

    helm rollback alloy <REVISION> -n logging

After rolling back, verify:

    kubectl get pod -n logging -o wide | grep alloy

    kubectl logs <alloy-pod-name> -n logging --tail=100

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

---

## Section Twenty-three: Uninstallation and Cleanup

### 23.1 Uninstall Alloy

    helm uninstall alloy -n logging

### 23.2 Check for残留 Resources

    kubectl get all -n logging

    kubectl get cm -n logging | grep alloy

    kubectl get sa -n logging | grep alloy

    kubectl get clusterrole | grep alloy

    kubectl get clusterrolebinding | grep alloy

### 23.3 Note

Uninstalling Alloy will not delete logs already stored in Loki.

To clear Loki data, it must be handled separately.

Do not arbitrarily delete Loki data in a production environment.

---

## Section Twenty-four: Practical Tasks

### 24.1 Task One: Verify Loki Availability

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

Acceptance criteria:

    [ ] The Loki Pod is running.
    [ ] The Loki Gateway Service exists.
    [ ] The /ready endpoint returns "ready".

### 24.2 Task Two: Prepare a Test Application

    kubectl create namespace app-demo

    kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo

    kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo

    kubectl patch deployment nginx-demo -n app-demo \
      --type='merge' \
      -p '{
        "spec": {
          "template": {
            "metadata": {
              "labels": {
                "app": "nginx-demo",
                "app.kubernetes.io/name": "nginx-demo",
                "team": "sre",
                "environment": "lab"
              }
            }
          }
        }
      }'

    kubectl rollout status deploy/nginx-demo -n app-demo

Acceptance criteria:

    [ ] The nginx-demo Pod is running.
    [ ] The nginx-demo Service exists.
    [ ] Endpoints are available.
    [ ] Logs can be viewed using kubectl logs.

### 24.3 Task Three: Deploy Alloy

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

    helm search repo grafana/alloy --versions | head -20

    helm show values grafana/alloy \
      --version <ALLOY_CHART_VERSION> \
      > values-alloy-default.yaml

    helm template alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml \
      > alloy-rendered.yaml

    helm install alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

Acceptance criteria:

    [ ] The Helm template execution is successful.
    [ ] Alloy is successfully installed.
    [ ] The Alloy DaemonSet has been created.
    [ ] The Alloy Pod is running.
    [ ] There are no persistent errors in the Alloy logs.

### 24.4 Task Four: Generate and Query Logs

Generate access logs:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    exit

Query labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Query app-demo logs:

    curl -G -s "http://12[ ] Nodes exist within the Loki labels.  
[ ] Clusters are also included in the Loki labels.  
[ ] It is possible to query logs by namespace.  
[ ] Logs can be searched by pod as well.  
[ ] Searching by container is supported.  
[ ] Nginx access logs can be queried.  
[ ] 404 error logs can be retrieved.  
[ ] The troubleshooting process for when Alloy fails to collect logs is understood.  

---

## Chapter 26: Common Misconceptions

### 26.1 Misconception 1: Deploying Loki automatically collects Pod logs  

Wrong.  

Loki serves as the server-side component.  

Additional collection agents are also required:  
    Alloy  
    Promtail  
    Fluent Bit  
    Filebeat  

In this document, only Alloy is deployed.  

### 26.2 Misconception 2: Alloy is a component of Loki  

Wrong.  

Alloy acts as the collection agent, while Loki handles log storage and querying.  

### 26.3 Misconception 3: Using DaemonSet mode does not limit the collection scope  

Not recommended.  

Each node with a DaemonSet installed has an Alloy agent.  

If every Alloy collects logs from all pods in the cluster, it may lead to duplicate data collection and increased API load.  

It is better to restrict collection to only the pods on the current node.  

### 26.4 Misconception 4: The more labels, the better  

Wrong.  

An excessive number of labels with a high cardinality can affect Loki’s performance.  
It is advisable to use labels judiciously.  

### 26.5 Misconception 5: If logs cannot be found, it must be due to a problem with Loki  

Not necessarily.  

Possible reasons include:  
    The application does not generate logs.  
    `kubectl logs` also returns no results.  
    Alloy lacks the necessary permissions.  
    Alloy has not covered all relevant nodes.  
    Incorrect configuration in Alloy.  
    Issues with the Loki Gateway.  
    Incorrect LogQL label formatting.  
    Incorrect query time range.  
    Logs have been filtered out.  

---

## Chapter 27: Summary

This document demonstrates how to use Grafana Alloy to collect Kubernetes Pod logs and forward them to Loki for storage and querying.  

The key components involved are:  
    `nginx-demo` Pod  
    `stdout / stderr` outputs from the Pod  
    `Alloy DaemonSet`  
    `discovery.kubernetes`  
    `discovery.relabel`  
    `loki.source.kubernetes`  
    `loki.process`  
    `loki.write`  
    `Loki Gateway`  
    `LogQL` queries  

With this setup, Loki no longer relies solely on manual data pushes and queries; it now has the capability to automatically collect Kubernetes logs.  

You can now perform queries such as:  
    `{namespace="app-demo"}`  
    `{namespace="app-demo", pod=~"nginx-demo-.*"}`  
    `{namespace="app-demo", container="nginx"}`  
    `{namespace="app-demo"} |= "GET"`  
    `{namespace="app-demo"} |= "404"`  

The key to mastering Alloy is not just installing it but understanding how to:  
    Identify which pods should be monitored.  
    Choose appropriate labels for logs.  
    Determine where the logs should be stored.  
    Prevent duplicate data collection.  
    Ensure proper access rights.  
    Enable efficient label-based queries.  
    Troubleshoot any collection failures.  

In the next chapter, we will explore:  
**07 - Loki Label Design and High Cardinality Issues**  

We will focus on understanding:  
- What labels are and how they work.  
- What streams are and their role in data processing.  
- Why `request_id` cannot be used as a label.  
- How to design labels for namespaces, pods, containers, applications, and teams.  
- How high cardinality labels can impact Loki’s performance.  
- How experiments can help analyze the effects of label design on queries and data writes.  

---

## References

- **Grafana Alloy Documentation**:  
  https://grafana.com/docs/alloy/latest/  

- **Deploying Grafana Alloy on Kubernetes**:  
  https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/  

- **Collecting Kubernetes Logs and Forwarding Them to Loki**:  
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/  

- **loki.write Component Documentation**:  
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/  

- **loki.source.kubernetes Component Documentation**:  
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/  

- **discovery.kubernetes