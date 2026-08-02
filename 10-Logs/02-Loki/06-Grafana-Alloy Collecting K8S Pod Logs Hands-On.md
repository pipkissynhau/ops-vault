# 06-Grafana-Alloy Collecting Kubernetes Pod Logs in Practice

## Document Explanation

This article is the sixth in the Loki special study series, used to deploy Grafana Alloy in a Kubernetes cluster and forward Kubernetes Pod logs to Loki after collection.

Previous articles have been completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-Instance Helm Deployment Practice
    05-Loki Object Storage Access MinIO Practice

The fourth article verified that the Loki server can manually write and query logs.

The fifth article connected the Loki backend storage to MinIO.

This article begins to connect the real Kubernetes Pod log collection chain:

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
    LogQL Query
      ↓
    Grafana Explore / Dashboard

This article focuses on answering the following questions:

- What is Alloy;
- What are the differences between Alloy and Promtail, Fluent Bit, Filebeat;
- Why new environments should prioritize Alloy;
- Why Alloy is usually run as a DaemonSet in Kubernetes;
- How to deploy Alloy via Helm;
- How to configure Alloy to discover Pods;
- How to add namespace, pod, container, node, app, cluster labels to logs;
- How to write Pod logs to Loki;
- How to verify if Alloy is running normally;
- How to verify if Loki has received Pod logs;
- How to use LogQL to query app-demo / nginx-demo logs;
- How to troubleshoot when Alloy cannot collect logs;
- What to pay attention to in production environments when Alloy collects Pod logs.

---

## Tags

#Loki #GrafanaAlloy #Kubernetes #PodLog #LogCollection #DaemonSet #LogQL #Grafana #SRE #Observation #LogDetachment

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/06-Grafana-Alloy Collecting K8S Pod Logs in Practice.md

---

## One, Experiment Objectives

After completing this article, you should be able to:

    1. Understand Alloy's position in the Loki log system.
    2. Use Helm to deploy Grafana Alloy.
    3. Deploy Alloy as a DaemonSet to Kubernetes nodes.
    4. Configure Alloy to discover Kubernetes Pods.
    5. Configure Alloy to add namespace / pod / container / node / app labels to logs.
    6. Configure Alloy to forward Pod logs to Loki.
    7. Verify if Alloy Pods cover all nodes.
    8. Verify if Alloy logs have no sending failures.
    9. Query logs in the app-demo Namespace in Loki.
    10. Query logs by pod / container / app in Loki.
    11. Generate logs through an Nginx test application.
    12. Master common troubleshooting methods for Alloy not collecting logs.
    13. Understand permission, label, performance, and security considerations for Alloy log collection in production environments.

---

## Two, Experiment Environment

### 2.1 Kubernetes Cluster

Experiment nodes:

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

This article mainly uses:

    logging:
        Deploy Loki, Alloy.

    app-demo:
        Deploy test applications and generate logs.

### 2.2 Completed Prerequisites

Already completed:

    [ ] Loki has been deployed
    [ ] Loki Gateway is accessible
    [ ] /ready returns ready
    [ ] /metrics is accessible
    [ ] Manual push/query succeeded
    [ ] MinIO integration is complete, or at least Loki server is available
    [ ] app-demo Namespace has been created

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

### 2.3 Tools

Required:

    kubectl
    helm
    curl
    jq
    grep

Check versions:

    kubectl version --client

    helm version

---

## Three, What is Alloy

Grafana Alloy is the next-generation log agent in the Grafana ecosystem.

It can collect:

    logs
    metrics
    traces
    profiles
    Kubernetes Pod logs
    Kubernetes Events
    system logs

In the Loki log system, Alloy is responsible for:

    Discovering log sources
    Reading logs
    Adding labels
    Processing logs
    Forwarding logs to Loki

Loki is responsible for:

    Receiving logs
    Storing logs
    Querying logs
    Log alerts

Grafana is responsible for:

    Query entry point
    Dashboard
    Explore
    Integration display of metrics and logs

AlertManager is responsible for:

    Alert notifications
    Alert grouping
    Alert suppression
    Alert silence

---

## Four, Differences Between Alloy, Promtail, Fluent Bit, and Filebeat

### 4.1 Alloy

Features: /think

# Grafana Ecosystem's Next-Generation Agent  
Can collect logs / metrics / traces.  
Integrated with Grafana components like Loki, Mimir, Tempo, etc.  
Recommended to learn and use first in new environments.  

**Suitable for:**  
- New Loki deployment  
- Unified log collection in Grafana ecosystem  
- Kubernetes Pod log collection  
- Future expansion of metrics / traces  

---

## 4.2 Promtail  

**Features:**  
- Loki's early log collection agent  
- Common in historical environments  
- Mainly for log collection to Loki  

**Notes:**  
- Not recommended to use Promtail as the long-term main solution in new environments  
- However, it's still necessary to understand, as many old clusters still use Promtail  

---

## 4.3 Fluent Bit  

**Features:**  
- Lightweight log collection and forwarding agent  
- Supports output to Loki, Elasticsearch, OpenSearch, Kafka, etc.  
- Very common in Kubernetes  

**Suitable for:**  
- Multi-backend log forwarding  
- Low resource consumption  
- Enterprise already has Fluent Bit standard  

---

## 4.4 Filebeat  

**Features:**  
- Log collector for Elastic ecosystem  
- Commonly used in ELK / EFK  

**Suitable for:**  
- Elasticsearch / Logstash / Kibana  
- Enterprise already has Elastic ecosystem  

---

## 4.5 This Series' Choice  

**Selected:**  
Grafana Alloy  
↓  
Loki  

**Reasons:**  
1. Closer to Grafana's official recommendation direction in new environments  
2. Naturally integrated with Loki  
3. Future expansion of Kubernetes Events, metrics, traces  
4. Suitable as the main learning path for cloud-native log systems  

---

## Five, Pod Log Collection Method Explanation  

Alloy commonly has two approaches to collect Kubernetes Pod logs.  

### 5.1 Method One: Tail Pod Logs via Kubernetes API  

**Pipeline:**  
Alloy  
↓  
Kubernetes API / Kubelet  
↓  
Pod logs  
↓  
Loki  

**Features:**  
- Doesn't necessarily need to read the host's /var/log/containers  
- Relatively intuitive configuration  
- Requires Kubernetes API permissions  
- Note API / kubelet pressure in large clusters  
- In DaemonSet mode, limit to collect only this node's Pods to avoid duplication  

This article's main approach uses this method:  
discovery.kubernetes  
discovery.relabel  
loki.source.kubernetes  
loki.process  
loki.write  

### 5.2 Method Two: Read Node Log Files  

**Pipeline:**  
/var/log/containers/*.log  
↓  
Alloy loki.source.file  
↓  
Loki  

**Features:**  
- Closer to traditional node log collection methods  
- Requires mounting the host's log directory  
- Needs to handle file paths and Kubernetes metadata  
- Higher permission requirements  
- Common in production environments  

This article first uses the official example's Kubernetes API method.  

Advanced steps can later supplement:  
local.file_match  
loki.source.file  
/var/log/containers  
/var/log/pods  

---

## Six, Overall Architecture  

The pipeline after deployment in this article:  

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
LogQL query  

**Architecture Diagram:**  

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

## Seven, Prepare Test Application  

If you've already deployed nginx-demo in Chapter 01, you can skip this section.

### 7.1 Create app-demo Namespace

    kubectl create namespace app-demo

If it already exists, you can ignore this step.

### 7.2 Deploy Nginx

    kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo

### 7.3 Add Recommended Labels to Deployment

Check the Deployment:

    kubectl get deploy nginx-demo -n app-demo --show-labels

Add labels to the Deployment template:

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

Wait for rolling update:

    kubectl rollout status deploy/nginx-demo -n app-demo

Check Pods:

    kubectl get pod -n app-demo -o wide --show-labels

### 7.4 Expose Service

If there is no Service, create one:

    kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo

Check:

    kubectl get svc -n app-demo

    kubectl get endpoints nginx-demo -n app-demo

### 7.5 Generate Logs

Start a temporary curl Pod:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    curl http://nginx-demo.app-demo.svc.cluster.local/healthz

Exit:

    exit

### 7.6 Verify with kubectl logs First

Check Pods:

    kubectl get pod -n app-demo

Check logs:

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

You should see Nginx access log entries.

If kubectl logs cannot retrieve logs, Alloy may also fail to collect logs.

---

## VIII. Prepare Helm Repository

Alloy uses the Grafana Helm repository.

### 8.1 Add Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 8.2 Search for Alloy Chart

    helm search repo grafana/alloy --versions | head -20

Record:

    Chart Version:
        <actual version>

It is recommended to install with a fixed version:

    --version <ALLOY_CHART_VERSION>

Do not use unversioned charts in production.

### 8.3 Export Default values

    helm show values grafana/alloy \
      --version <ALLOY_CHART_VERSION> \
      > values-alloy-default.yaml

Search for key fields:

    grep -n "controller:" values-alloy-default.yaml

    grep -n "type:" values-alloy-default.yaml

    grep -n "configMap:" values-alloy-default.yaml

    grep -n "mounts:" values-alloy-default.yaml

    grep -n "varlog" values-alloy-default.yaml

    grep -n "rbac:" values-alloy-default.yaml

---

## IX. Write Alloy values File

### 9.1 Create values File

Create:

    values-alloy-loki.yaml

### 9.2 Content of values-alloy-loki.yaml

Note:

    This configuration uses Kubernetes API to collect Pod logs.
    Alloy runs as a DaemonSet.
    discovery.kubernetes uses field selector to limit collection to current node Pods, avoiding DaemonSet multi-replica duplication.
    loki.write points to Loki Gateway.
    Loki currently has auth_enabled=false, so tenant_id is not configured.
    If production Loki enables multi-tenancy, you need to add tenant_id or authentication configuration.

Content:

    alloy:
      mounts:
        varlog: true
        dockercontainers: false

      configMap:
        content: |
          logging {
            level  = "info"
            format = "logfmt"
          }

          discovery.kubernetes "pod" {
            role = "pod"

            selectors {
              role  = "pod"
              field = "spec.nodeName=" + coalesce(sys.env("HOSTNAME"), constants.hostname)
            }
          }

          discovery.relabel "pod_logs" {
            targets = discovery.kubernetes.pod.targets

            rule {
              source_labels = ["__meta_kubernetes_namespace"]
              action        = "replace"
              target_label  = "namespace"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_name"]
              action        = "replace"
              target_label  = "pod"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_container_name"]
              action        = "replace"
              target_label  = "container"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_node_name"]
              action        = "replace"
              target_label  = "node"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
              action        = "replace"
              target_label  = "app"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_label_app"]
              action        = "replace"
              target_label  = "app"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_label_team"]
              action        = "replace"
              target_label  = "team"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_label_environment"]
              action        = "replace"
              target_label  = "environment"
            }

            rule {
              source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_container_name"]
              action        = "replace"
              target_label  = "job"
              separator     = "/"
              replacement   = "$1"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
              action        = "replace"
              target_label  = "__path__"
              separator     = "/"
              replacement   = "/var/log/pods/*$1/*.log"
            }

            rule {
              source_labels = ["__meta_kubernetes_pod_container_id"]
              action        = "replace"
              target_label  = "container_runtime"
              regex         = `^(\S+):\/\/.+$`
              replacement   = "$1"
            }
          }

          loki.source.kubernetes "pod_logs" {
            targets    = discovery.relabel.pod_logs.output
            forward_to = [loki.process.pod_logs.receiver]
          }

          loki.process "pod_logs" {
            stage.static_labels {
              values = {
                cluster = "k8s-lab",
              }
            }

            forward_to = [loki.write.loki.receiver]
          }

```yaml
loki.write "loki" {
  endpoint {
    url = "http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"
  }
}

controller:
  type: daemonset

serviceAccount:
  create: true

rbac:
  create: true
```

### 9.3 Configuration Explanation

    controller.type: daemonset

Indicates that an Alloy Pod runs on each node.

Suitable for collecting logs from Pods on this node.

    loki.write "loki"

Defines the writing target.

    url = "http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"

Indicates that logs are pushed to Loki Gateway.

    discovery.kubernetes "pod"

Discovers Kubernetes Pods.

    selectors field = spec.nodeName=...

Limits the current Alloy to collect only Pods on this node, avoiding duplicate collection of all cluster Pods in DaemonSet mode.

    discovery.relabel

Converts Kubernetes metadata to Loki labels.

    loki.source.kubernetes

Reads logs from Kubernetes Pod targets.

    loki.process

Processes logs and adds static labels.

    cluster = "k8s-lab"

Used to identify which cluster the logs come from.

### 9.4 About varlog Mounting

The main approach used in this document is:

    loki.source.kubernetes

This method tails Pod logs via Kubernetes API.

The values still retain:

    alloy.mounts.varlog: true

Reasons:

    1. Facilitates future expansion of loki.source.file.
    2. Facilitates troubleshooting node log paths.
    3. Aligns with common file collection methods in production environments.

If strict permission control is required, you can disable varlog mounting in API collection mode.

---

## TenI don't know.Installation of Alloy

### 10.1 Helm Template Check

First execute template:

    helm template alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml \
      > alloy-rendered.yaml

Check resources:

    grep "^kind:" alloy-rendered.yaml | sort | uniq -c

Check DaemonSet:

    grep -n "kind: DaemonSet" alloy-rendered.yaml

Check ConfigMap:

    grep -n "kind: ConfigMap" alloy-rendered.yaml

Check RBAC:

    grep -n "kind: ClusterRole" alloy-rendered.yaml

    grep -n "kind: ClusterRoleBinding" alloy-rendered.yaml

Check image:

    grep -n "image:" alloy-rendered.yaml | sort -u

### 10.2 Installation

    helm install alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

### 10.3 View Release

    helm list -n logging

Expected:

    alloy   logging   deployed

### 10.4 View Alloy Pod

    kubectl get pods -n logging -o wide | grep alloy

Or:

    kubectl get ds -n logging

Expected:

    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
    Node count      Node count      Node count

If your cluster has 3 nodes and the DaemonSet doesn't restrict master nodes, there may be 3 Alloy Pods.

If master nodes have taints and Alloy lacks toleration, they may only run on worker nodes.

---

## ElevenI don't know.Verify Alloy Permissions

### 11.1 View ServiceAccount

    kubectl get sa -n logging | grep alloy

### 11.2 View RBAC

    kubectl get clusterrole | grep alloy

    kubectl get clusterrolebinding | grep alloy

### 11.3 Verify Pod Read Access

Assuming the ServiceAccount name is alloy:

    kubectl auth can-i list pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i watch pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i get pods/log \
      --as=system:serviceaccount:logging:alloy

If it returns no, check the RBAC configuration in the Helm Chart.

---

## TwelveI don't know.View Alloy Runtime Logs

### 12.1 View Alloy Pod

    kubectl get pod -n logging -o wide | grep alloy

### 12.2 View Alloy Logs

    kubectl logs <alloy-pod-name> -n logging --tail=200

If there are multiple containers:

    kubectl get pod <alloy-pod-name> -n logging -o jsonpath='{.spec.containers[*].name}'

Then specify the container:

kubectl logs <alloy-pod-name> -n logging -c alloy --tail=200

### 12.3 Filtering Errors

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|loki|push|kubernetes|permission|denied|timeout"

### 12.4 Normal Direction

Normally, you should see:

    Alloy started successfully
    Configuration loaded successfully
    Components running
    No persistent error
    No failed Loki sends
    No RBAC permission denied

### 12.5 Abnormal Direction

Common errors:

    cannot list resource pods
    forbidden
    connection refused
    no such host
    context deadline exceeded
    failed to send batch
    404 Not Found
    429 Too Many Requests
    500 Internal Server Error
    unknown escape sequence
    invalid component configuration

---

## ThirteenI don't know.Verifying if Loki Has Received Logs

### 13.1 Port Forward Loki Gateway

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

### 13.2 Query Label Names

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Expected to gradually see:

    namespace
    pod
    container
    node
    app
    job
    cluster

If only the 04/05th manually written labels appear, it may indicate Alloy logs haven't been successfully written yet.

### 13.3 Query namespace Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

Expected to include:

    app-demo
    logging
    kube-system

Depending on the scrape range and Pods in the cluster.

### 13.4 Query app-demo Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

### 13.5 Query nginx-demo Logs

If the app label exists:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode 'limit=20' | jq

If the app label doesn't appear, you can first query by pod:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

Then:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"}' \
      --data-urlencode 'limit=20' | jq

---

## FourteenI don't know.Generating Logs and Querying

### 14.1 Revisit Nginx

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    curl http://nginx-demo.app-demo.svc.cluster.local/api/test

Exit:

    exit

### 14.2 kubectl logs Verification

Check nginx Pod:

    kubectl get pod -n app-demo

Check logs:

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

Expected to see:

    GET /
    GET /notfound
    GET /api/test

### 14.3 Loki Query All app-demo Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=50' | jq

### 14.4 Query nginx-demo Access Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "GET"' \
      --data-urlencode 'limit=50' | jq

### 14.5 Query 404 Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"' \
      --data-urlencode 'limit=50' | jq

### 14.6 Query Logs of a Specific Container

First check container labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/container/values" | jq

Query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=50' | jq

---

## FifteenI don't know.Using Grafana Explore to Query

If Grafana is already deployed and the Loki Data Source is added, you can directly query in Grafana Explore.

### 15.1 Query app-demo

LogQL:

    {namespace="app-demo"}

### 15.2 Query nginx-demo

### 15.3 Querying GET Requests

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "GET"

### 15.4 Querying 404

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"

### 15.5 Querying Non-Health Checks

    {namespace="app-demo"} != "/healthz"

### 15.6 Querying ERROR / Timeout / Exception

    {namespace="app-demo"} |~ "(?i)error|timeout|exception|panic|traceback"

**Note:**

    Nginx examples may not necessarily contain ERROR.
    The 09th chapter will deploy a JSON log demo application for advanced LogQL practice.

---

## SixteenI don't know.Alloy Configuration Update Methods

### 16.1 Modifying Values

Edit:

    values-alloy-loki.yaml

Render after modification:

    helm template alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml \
      > alloy-rendered.yaml

### 16.2 Executing Upgrade

    helm upgrade alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

### 16.3 Viewing Rolling Updates

    kubectl rollout status ds/alloy -n logging

If the DaemonSet name is not "alloy":

    kubectl get ds -n logging

Then execute the corresponding name.

### 16.4 Checking if New Configuration is Effective

    kubectl get cm -n logging | grep alloy

    kubectl describe pod <alloy-pod-name> -n logging

    kubectl logs <alloy-pod-name> -n logging --tail=200

### 16.5 Backing Up Configuration

    helm get values alloy -n logging -a > backup-values-alloy.yaml

**Production Recommendations:**

    values-alloy-loki.yaml should be included in Git.
    Perform diff before and after changes.
    Avoid direct kubectl edit ConfigMap.

---

## SeventeenI don't know.Log Collection Scope Control

### 17.1 Risks of Default Full Namespace Collection

If collecting logs from all Pods, it may include:

    kube-system
    monitoring
    logging
    ingress-nginx
    Database logs
    Business logs
    Debug logs

**Risks:**

    Excessive log volume
    Rapid storage growth
    Slower queries
    Sensitive logs entering Loki
    Increased alert noise

### 17.2 Collecting Only Specified Namespaces

You can retain specified Namespaces in discovery.relabel.

**Example:** Retain only "app-demo" and "ai-prod".

Add the following to discovery.relabel:

    rule {
      source_labels = ["__meta_kubernetes_namespace"]
      action        = "keep"
      regex         = "app-demo|ai-prod"
    }

**Note:**

    The order of keep/drop rules is important.
    It is recommended to keep first, then perform label replace.
    A helm upgrade is required after modification.

### 17.3 Excluding kube-system

If you don't want to collect kube-system:

    rule {
      source_labels = ["__meta_kubernetes_namespace"]
      action        = "drop"
      regex         = "kube-system"
    }

### 17.4 Excluding Alloy's Own Logs

If you don't want to collect Alloy's own logs in the logging Namespace:

    rule {
      source_labels = ["__meta_kubernetes_namespace"]
      action        = "drop"
      regex         = "logging"
    }

**Note for Learning Phase:**

    It is recommended not to filter too early.
    Reason:
      You can see Alloy / Loki's own logs.
      It helps verify if collection is effective.

Adjust according to log volume and management policies in production.

---

## EighteenI don't know.Label Design Recommendations

### 18.1 Recommended Labels to Keep

**Recommended:**

    cluster
    namespace
    pod
    container
    node
    app
    team
    environment
    job

These labels are suitable for:

    Query range filtering
    Dashboard variables
    Alert routing
    Team ownership
    Multi-cluster differentiation

### 18.2 Not Recommended as Loki Labels

**Not Recommended:**

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message
    stacktrace
    ip
    container_id (full value)

**Reason:**

    High cardinality.
    Generates a large number of streams.
    Increases index pressure.
    Affects write and query performance.

### 18.3 How to Handle High Cardinality Fields

These fields can remain in the log body.

Use content filtering for queries:

    {namespace="app-demo"} |= "request_id=abc"

Or JSON parsing:

    {namespace="app-demo"} | json | trace_id="abc"

Next, the 07th article will specifically experiment:

    Loki Label Design and High Cardinality Problem Experiment

---

## NineteenI don't know.Alloy Unable to Collect Logs Troubleshooting

### 19.1 Fault Phenomenon One: kubectl logs has logs, but Loki cannot find them

Troubleshooting order:

    1. Is the Alloy Pod Running.
    2. Does the Alloy DaemonSet cover the Pod's node.
    3. Does Alloy have pods/log permissions.
    4. Is the Alloy configuration loaded successfully.
    5. Is the loki.write URL correct.
    6. Is the Loki Gateway accessible.
    7. Is the Loki /ready ready.
    8. Does Alloy logs have failed to send batch.
    9. Does Loki logs have rejected write.
    10. Are the LogQL labels written incorrectly.
    11. Is the query time range correct.

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

Common causes:

    The node has taint.
    Alloy has no toleration.
    nodeSelector restrictions.
    Resource insufficiency.
    Image pull failure.
    PodSecurity or security policy restrictions.

Handling:

    Add tolerations to Alloy.
    Check nodeSelector.
    Check Events.
    Check image repository access.

### 19.3 Fault Phenomenon Three: Alloy has insufficient permissions

Logs may show:

    forbidden
    cannot list resource pods
    cannot get resource pods/log

Check:

    kubectl auth can-i list pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i get pods/log \
      --as=system:serviceaccount:logging:alloy

Handling:

    Confirm rbac.create=true.
    Confirm ServiceAccount name is correct.
    Confirm ClusterRole / ClusterRoleBinding creation succeeded.

### 19.4 Fault Phenomenon Four: Alloy fails to connect to Loki

Alloy logs may show:

    connection refused
    no such host
    context deadline exceeded
    failed to send batch

Check Loki Gateway:

    kubectl get svc -n logging

    kubectl get endpoints loki-gateway -n logging

Test from logging Namespace:

    kubectl run curl-loki-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside container:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

Exit:

    exit

### 19.5 Fault Phenomenon Five: LogQL labels cannot be found

First check labels:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Then check namespace:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

Then check pod:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/pod/values | jq

If no pod/container:

    discovery.relabel rule may not take effect.
    Alloy configuration may not be updated.
    Query time range may be incorrect.
    Loki may not have new logs yet.

### 19.6 Fault Phenomenon Six: unknown escape sequence

If Alloy configuration has regex errors, may see:

    unknown escape sequence

Cause:

    \S, \d, etc. in regex need escaping in double quotes.
    Recommend using raw string with backticks.

Correct:

    regex = `^(\S+):\/\/.+$`

Not recommended:

    regex = "^(\S+):\/\/.+$"

---

## TwentyI don't know.Loki-side Troubleshooting

### 20.1 Check if Loki is ready

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

### 20.2 Check Loki logs

    kubectl logs <loki-pod-name> -n logging --tail=300

Filter:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|push|ingest|stream|limit|429|500"

### 20.3 Common reasons for Loki rejecting writes

    ingestion rate limit
    too many streams
    label name invalid
    label value too long
    entry too far behind
    entry too far in future
    line too long
    tenant missing
    auth failed

### 20.4 Check Loki labels

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

### 20.5 Check Loki metrics

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

---

## 21. Production Considerations

### 21.1 Alloy Resource Limits

Resources should be set in production.

Example direction:

    alloy:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi

Notes:

    The actual fields are based on helm show values grafana/alloy.
    Increase resources when log volume is large.
    Do not set no limits, nor set too low limits.

### 21.2 Log Collection Scope Should Be Governed

It is not recommended to collect all logs in production blindly.

Recommendations:

    Grade by Namespace.
    Filter high-frequency low-value logs.
    Filter health check logs.
    Control DEBUG logs.
    Special governance for high log volume applications.
    Separate strategies for sensitive Namespaces.

### 21.3 Do Not Log Sensitive Information

Prohibited outputs from application side:

    Password
    Token
    Access key
    Secret key
    Authorization header
    Cookie
    Private key
    Plain text phone number
    ID card
    Bank card
    Database connection string password

Principles:

    Application side not outputting sensitive information is the top priority.
    Data desensitization on Alloy side can only be used as a supplement.
    Do not rely on log platform for post-cleaning sensitive information.

### 21.4 Avoid Excessive Labels

Recommended labels:

    cluster
    namespace
    pod
    container
    node
    app
    team
    environment

Do not use high cardinality fields as labels:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message
    stacktrace

### 21.5 Monitor Alloy Itself

Need to monitor:

    Alloy Pod status
    Alloy DaemonSet coverage of nodes
    Alloy sending failures
    Alloy restart count
    Alloy CPU / Memory
    Alloy log errors
    Loki write failures
    Loki 429
    Loki 5xx

### 21.6 Changes to Collection Endpoints Should Go Through Git

Recommendations:

    values-alloy-loki.yaml should be included in Git.
    Helm template before changes.
    Helm upgrade after changes.
    Verify labels and logs after changes.
    Keep rollback versions.

---

## 22. Upgrade and Rollback

### 22.1 Check Release

    helm list -n logging

    helm status alloy -n logging

    helm history alloy -n logging

### 22.2 Backup Values

    helm get values alloy -n logging -a > backup-values-alloy.yaml

### 22.3 Upgrade Alloy

    helm upgrade alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

### 22.4 Check DaemonSet Update

    kubectl rollout status ds/alloy -n logging

If the name is different:

    kubectl get ds -n logging

### 22.5 Rollback

    helm rollback alloy <REVISION> -n logging

After rollback verification:

    kubectl get pod -n logging -o wide | grep alloy

    kubectl logs <alloy-pod-name> -n logging --tail=100

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

---

## 23. Uninstallation and Cleanup

### 23.1 Uninstall Alloy

    helm uninstall alloy -n logging

### 23.2 Check for Residual Resources

    kubectl get all -n logging

    kubectl get cm -n logging | grep alloy

    kubectl get sa -n logging | grep alloy

    kubectl get clusterrole | grep alloy

    kubectl get clusterrolebinding | grep alloy

### 23.3 Notes

Uninstalling Alloy will not delete logs already written to Loki.

If you want to clean up Loki data, you need to handle Loki storage separately.

Do not arbitrarily delete Loki data in production environments.

---

## 24. Hands-on Tasks

### 24.1 Task 1: Confirm Loki Availability

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

Acceptance criteria:

    [ ] Loki Pod Running
    [ ] Loki Gateway Service exists
    [ ] /ready returns ready

### 24.2 Task 2: Prepare Test Application

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

Verification:

    [ ] nginx-demo Pod Running
    [ ] nginx-demo Service exists
    [ ] Endpoints not empty
    [ ] kubectl logs can view logs

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

Verification:

    [ ] helm template no errors
    [ ] helm install successful
    [ ] Alloy DaemonSet created
    [ ] Alloy Pod Running
    [ ] Alloy logs no persistent errors

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

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Query 404:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"' \
      --data-urlencode 'limit=20' | jq

Verification:

    [ ] labels can see namespace
    [ ] label/namespace/values can see app-demo
    [ ] can query nginx-demo logs
    [ ] can query GET requests
    [ ] can query 404 requests

---

## Twenty-fiveI don't know.Verification Checklist

After completing this document, confirm:

    [ ] Loki Gateway accessible
    [ ] Loki /ready returns ready
    [ ] app-demo Namespace exists
    [ ] nginx-demo Pod Running
    [ ] nginx-demo Service normal
    [ ] kubectl logs can view nginx-demo logs
    [ ] grafana Helm repository added
    [ ] Alloy Chart version fixed
    [ ] values-alloy-loki.yaml written
    [ ] helm template passed
    [ ] Alloy Helm Release deployed
    [ ] Alloy DaemonSet created
    [ ] Alloy Pod covers expected nodes
    [ ] Alloy Pod logs no persistent errors
    [ ] Alloy has pods / pods/log permissions
    [ ] Loki labels contains namespace
    [ ] Loki labels contains pod
    [ ] Loki labels contains container
    [ ] Loki labels contains node
    [ ] Loki labels contains cluster
    [ ] can query logs by namespace
    [ ] can query logs by pod
    [ ] can query logs by container
    [ ] can query Nginx access logs
    [ ] can query 404 logs
    [ ] understand troubleshooting path for Alloy not collecting logs

---

## Twenty-sixI don't know.Common Misconceptions

### 26.1 Misconception One: After deploying Loki, Pod logs can be automatically collected

Error.

Loki is a server-side component.

A collector is still needed:

    Alloy
    Promtail
    Fluent Bit
    Filebeat

This document deploys:

    Alloy

### 26.2 Misconception Two: Alloy is a component of Loki

Error.

Alloy is a collector agent.

Loki is a log storage and query server.

### 26.3 Mistake Three: DaemonSet mode can be unrestricted in collection scope

Not recommended.

DaemonSet has Alloy on each node.

If each Alloy collects all cluster Pods, it may result in duplicate collection and increased API pressure.

Should be limited as much as possible:

    Collect only the Pods on the current node

### 26.4 Mistake Four: More labels are better

Error.

Too many labels and high cardinality can affect Loki's performance.

Recommend to be cautious with labels.

### 26.5 Mistake Five: Not finding logs doesn't necessarily mean Loki is broken

Not necessarily.

Possible reasons:

    The application has no logs
    kubectl logs also has none
    Alloy lacks permissions
    Alloy hasn't covered the nodes
    Alloy configuration is incorrect
    Loki Gateway is unreachable
    LogQL labels are written incorrectly
    Query time range is incorrect
    Logs are filtered

---

## Twenty-Seven, Summary

This article completes the practical implementation of Grafana Alloy collecting Kubernetes Pod logs and forwarding them to Loki.

Core pipeline:

    nginx-demo Pod
      ↓
    stdout / stderr
      ↓
    Alloy DaemonSet
      ↓
    discovery.kubernetes
      ↓
    discovery.relabel
      ↓
    loki.source.kubernetes
      ↓
    loki.process
      ↓
    loki.write
      ↓
    Loki Gateway
      ↓
    Loki
      ↓
    LogQL query

After completing this article, Loki is no longer just capable of manual push/query, but now has real Kubernetes log collection capabilities.

Now you can query:

    {namespace="app-demo"}

    {namespace="app-demo", pod=~"nginx-demo-.*"}

    {namespace="app-demo", container="nginx"}

    {namespace="app-demo"} |= "GET"

    {namespace="app-demo"} |= "404"

Mastering Alloy's key is not just installation, but understanding:

    Which Pods to discover
    Which labels to add
    Which Loki to send to
    Whether collection is duplicated
    Whether there are permissions
    Whether querying by labels is possible
    Whether troubleshooting collection failures is possible

Next article will enter:

    07-Loki label design and high cardinality experiment

Key learning points:

    What is a label
    What is a stream
    Why request_id cannot be a label
    How to design namespace / pod / container / app / team labels
    How high cardinality can cripple Loki
    How to observe the impact of label design on queries and writes through experiments

---

## Reference Documents

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- Deploy Grafana Alloy on Kubernetes:
  https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/

- Collect Kubernetes logs and forward them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- loki.write component:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/

- loki.source.kubernetes component:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- discovery.kubernetes component:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.kubernetes/

- discovery.relabel component:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/

- loki.process component:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.process/

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Get started with Grafana Loki:
  https://grafana.com/docs/loki/latest/get-started/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes kubectl logs:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- Helm Documentation:
  https://helm.sh/docs/