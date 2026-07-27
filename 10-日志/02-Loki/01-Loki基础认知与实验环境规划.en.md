# 01-Loki Basic Understanding and Experimental Environment Planning

## Document Description

This document is the first in a series dedicated to learning about Loki, aiming to establish a fundamental understanding of the Loki logging system and plan the Kubernetes environment, Namespace, components, access methods, storage options, and verification paths required for the subsequent 15 practical experiments with Loki.

This document does not directly delve into Helm deployment but first addresses the following questions:

- What is Loki?
- Why does Kubernetes need a centralized logging system?
- How does Loki differ from ELK/EFK?
- What are the roles of Prometheus, Loki, Grafana, and Alloy?
- Where do Pod logs originate from?
- In a containerd environment, where are Pod logs typically stored?
- What is the difference between `kubectl logs` and a centralized logging platform?
- Why is it recommended to use Grafana Alloy over Promtail in new environments?
- What components are needed for subsequent Loki experiments?
- How should directories, Namespaces, access points, storage, and verification methods be planned?

This document serves as the foundation for the upcoming series of practical Loki experiments.

---

## Tags

#Loki #Grafana #GrafanaAlloy #Kubernetes #PodLogs #LogCollection #LogQL #SRE #Observability #LoggingSystem

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/01-LokiBasicUnderstandingAndExperimentalEnvironmentPlanning.md

---

## I. What is Loki

Loki is a logging aggregation system within the Grafana ecosystem.

Its primary functions include:

- Collecting logs;
- Storing logs;
- Querying logs;
- Displaying logs in Grafana;
- Generating alerts based on logs;
- Integrating with Prometheus metrics for troubleshooting purposes.

In a Kubernetes environment, Loki is commonly used as follows:

    Kubernetes Pod
      ↓
    stdout / stderr
      ↓
    Node log files
      ↓
    Grafana Alloy / Promtail / Fluent Bit
      ↓
    Loki
      ↓
    Grafana Explore / Dashboard
      ↓
    LogQL queries / log alerts / troubleshooting analysis

In simple terms:

- Prometheus is responsible for metrics.
- Loki handles logs.
- Grafana serves as the display interface.
- AlertManager handles notifications.
- Alloy is used for log collection and forwarding.

Loki is particularly suitable for Kubernetes operations, SRE tasks, cloud-native platforms, and GPU/AI workload troubleshooting scenarios.

---

## II. Why Does Kubernetes Need a Centralized Logging System

In traditional server environments, logging troubleshooting typically involves:

    ssh login to the server
    navigate to `/var/log`
    monitor `app.log` in real-time
    search for errors using `grep`

However, this approach becomes ineffective in Kubernetes for several reasons:

- Pods are dynamically created and destroyed.
- Pods may be scheduled on different Nodes.
- Old container logs may be lost or rotated after a Pod restart.
- An application often has multiple replicas.
- A service may span multiple Namespaces.
- Local logs may become inaccessible after a node failure.
- In shared clusters, it is impractical for everyone to log in to nodes to view logs.
- `kubectl logs` is only suitable for temporary viewing of individual Pods and not for long-term retrieval.
- Production environments require unified logging queries by Namespace, Pod, Container, App, Node, and time range.
- Logs are needed for alerts, post-mortem analysis, auditing, and issue tracking.

Therefore, production Kubernetes environments typically require:

    Application output (stdout/stderr)
      ↓
    Container runtime logs written to nodes
      ↓
    Log collection agents that collect these logs
      ↓
    Centralized logging systems for storage
      ↓
    Grafana/Kibana for querying and visualization
      ↓
    Log alerts, fault troubleshooting, and analysis

---

## III. How Does Loki Differ from ELK/EFK

### 3.1 What are ELK/EFK

ELK typically refers to:

    Elasticsearch
    Logstash
    Kibana

EFK typically refers to:

    Elasticsearch
    Fluentd/Fluent Bit
    Kibana

Alternatively, OpenSearch and Fluent Bit with OpenSearch Dashboards can also be used.

ELK/EFK are more akin to full-text search logging systems. Their key features include:

- Support for full-text searches;
- Strong field query capabilities;
- Advanced aggregation and analysis tools;
- Mature Kibana query interface;
- Suitable for auditing, security, and business log retrieval;
- Relatively higher resource consumption;
- More complex index, shard, and lifecycle management.

### 3.2 Loki’s Features

Loki’s core features include:

- Indexing log labels by default but not the log content itself.
- This means that Loki focuses on:
    - Narrowing down search results using labels first
    - Then searching for specific keywords within the log content.
    For exampleIf it is indeed necessary to write files for business purposes, a separate log collection agent should be configured to collect data from that path.

### 5.2 Log Paths in the containerd Environment

Common paths in the containerd environment:

    /var/log/containers/
    /var/log/pods/

Common file formats:

    /var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

To view the log path of a node:

    ls -l /var/log/containers/
    ls -l /var/log/pods/

These paths are the primary targets for log agents such as Alloy, Fluent Bit, and Filebeat.

### 5.3 kubectl logs

To view the current logs of a Pod:

    kubectl logs <pod-name> -n <namespace>

To view the logs of the last container instance:

    kubectl logs <pod-name> -n <namespace> --previous

To view the logs of a specific container:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

To view the last 100 lines:

    kubectl logs <pod-name> -n <namespace> --tail=100

For continuous monitoring:

    kubectl logs <pod-name> -n <namespace> -f

To view logs from the past 10 minutes:

    kubectl logs <pod-name> -n <namespace> --since=10m

### 5.4 Limitations of kubectl logs

kubectl logs is very suitable for temporary troubleshooting.

However, it is not appropriate for use in a production log system.

Limitations:

- Not suitable for long-term log retention;
- Not suitable for searching across multiple Pods;
- Not suitable for searching across different Namespaces;
- Not suitable for aggregating logs by keyword;
- Not suitable for building dashboards;
- Not suitable for generating log alerts;
- Logs may become unavailable after the Pod is deleted;
- Logs may become unavailable in the event of a node failure;
- Troubleshooting efficiency is lower in multi-replica services.

Therefore, a centralized log system like Loki is needed.🔤 Add the Grafana chart repository:  
`helm repo add grafana https://grafana.github.io/helm-charts`

Update the repository:  
`helm repo update`

To view Loki charts:  
`helm search repo grafana/loki --versions`

To view Alloy charts:  
`helm search repo grafana/alloy --versions`

To view Grafana charts:  
`helm search repo grafana/grafana --versions`

### 9.3 Check the storage class  

View the StorageClass:  
`kubectl get storageclass`

If there is no default StorageClass, you will need to handle Loki PVCs or MinIO PVCs separately later on.

For learning purposes, you can use the following options initially:  
`hostPath`, `local-path-provisioner`, `NFS StorageClass`, `Longhorn`, or manual PV/PVC configuration.

In a production environment, it is recommended to use reliable block storage or object storage solutions.

### 9.4 Check the node log paths  

Execute the following commands on any Worker node:  
`ls -l /var/log/containers/`  
`ls -l /var/log/pods/`

If these paths exist, it means that the container log paths meet the requirements for subsequent log collection.

If the paths do not exist, you need to verify the following:  
- Whether Kubernetes is running Pods normally.  
- Whether the containerd configuration is correct.  
- Whether there are any business Pods already on the node.  
- Whether the log paths have been cleared or due to system differences.

### 9.5 Create a test application namespace  

Create the `app-demo` namespace:  
`kubectl create namespace app-demo`

If it already exists, you will receive a message indicating "Already Exists"; you can ignore this warning.

Verify its existence:  
`kubectl get ns app-demo`

---

## Section 10: Prepare a test application that generates logs  

For further learning on Loki, you will need a stable source of logs. You can start by deploying a simple Nginx server as a basic log source.

### 10.1 Deploy Nginx  

Create a deployment named `nginx-demo`:  
`kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo`

Expose the service:  
`kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo`

Verify the deployment and service:  
`kubectl get pod -n app-demo -o wide`  
`kubectl get svc -n app-demo`

### 10.2 Access Nginx to generate logs  

Create a temporary `curl-test` Pod:  
`kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      --sh`

Inside the `curl-test` container, execute the following commands to access Nginx and generate logs:  
`curl http://nginx-demo.app-demo.svc.cluster.local`  
`curl http://nginx-demo.app-demo.svc.cluster.local/notfound`  

Exit the container:  
`exit`

### 10.3 View the logs using `kubectl logs`  

View the Pod:  
`kubectl get pod -n app-demo`

To view the logs in detail, use the following command:  
`kubectl logs <nginx-pod-name> -n app-demo --tail=50`

If you can see the access logs, it means that:  
- The application is logging messages normally.  
- `kubectl logs` is functioning correctly.  
- The log collection agent will be able to retrieve these logs later on.

---

## Section 11: Prepare a test application that outputs error logs  

Nginx access logs are suitable for basic verification, but they are not ideal for advanced LogQL queries. Later on, you will need a demo application that can output the following types of logs:  
- INFO logs  
- WARN logs  
- ERROR logs  
- JSON logs  
- A `status` field  
- A `duration_ms` field  
- A `timeout` field  
- An `exception` field  

An example log format is as follows:  
```json
{
  "timestamp": "2026-04-30T12:00:00+08:00",
  "level": "info",
  "service": "app-demo",
  "msg": "Request successful",
  "status": 200,
  "duration_ms": 32
}
```
```json
{
  "timestamp": "2026-04-30T12:00:03+08:00",
  "level": "error",
  "service": "app-demo",
  "msg":---  

## Section Fourteen: Experimental Risks and Production Considerations

### 14.1 Do Not Directly Copy Learning Configurations to Production

The learning environment can include:

    Single-node mode
    Port forwarding
    Simple PVC configuration
    Default resources
    Simple authentication
    Short-term data retention

The production environment should consider:

    High availability
    Object storage
    Data retention periods
    Resource requests and limits
    Query restrictions
    Write throttling
    Tenant isolation
    HTTPS
    Authentication
    Alerts
    Backup
    Auditing

### 14.2 Logs May Contain Sensitive Information

Production logs must avoid including:

    Passwords
    Tokens
    Access keys
    Secret keys
    Authorization headers
    Cookies
    Private keys
    Plain text phone numbers
    Identity cards
    Bank card details
    Database connection string passwords

Principles:

    Preventing sensitive information from being displayed on the application side is the top priority.
    Data desensitization at the collection stage is a supplementary measure.
    Do not rely on the log platform to clean up sensitive data afterwards.

### 14.3 Log Volume Needs to Be Managed

More logs do not necessarily mean better performance.

Excessive log volume can lead to:

- Increased storage costs;
- Higher writing pressure on Loki;
- Slower query performance;
- More alert noise;
- Higher resource consumption by collection agents;
- Increased network bandwidth usage.

Production recommendations include:

    Controlling the amount of DEBUG logs.
    Filtering out health check logs.
    Limiting the output of large fields.
    Setting appropriate retention periods.
    Managing applications with high log volumes effectively.
    Regularly analyzing the top sources of logs.

---  

## Section Fifteen: Experimental Tasks in This Chapter

In this chapter, Loki will not be deployed; only environmental setup and basic verifications will be performed.

### 15.1 Create Namespaces

    `kubectl create namespace logging`
    `kubectl create namespace monitoring`
    `kubectl create namespace app-demo`
    `kubectl create namespace minio`

Verification:

    `kubectl get ns`

### 15.2 Add the Grafana Helm Repository

    `helm repo add grafana https://grafana.github.io/helm-charts`

    `helm repo update`

Verification:

    `helm search repo grafana/loki --versions`
    `helm search repo grafana/alloy --versions`
    `helm search repo grafana/grafana --versions`

### 15.3 Deploy a Nginx Test Application

    `kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo`

    `kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo`

View results:

    `kubectl get pod -n app-demo -o wide`
    `kubectl get svc -n app-demo`

### 15.4 Generate Logs

    `kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh`

Inside the container, execute:

    `curl http://nginx-demo.app-demo.svc.cluster.local`
    `curl http://nginx-demo.app-demo.svc.cluster.local/notfound`

Exit the container:

    `exit`

### 15.5 View Pod Logs

View the Pod:

    `kubectl get pod -n app-demo`

View logs:

    `kubectl logs <nginx-pod-name> -n app-demo --tail=50`

Expected result:

    The Nginx access logs should be visible.

### 15.6 View Node Log Paths

On the node where the Pod is located, execute:

    `ls -l /var/log/containers/ | grep nginx-demo`
    `ls -l /var/log/pods/ | grep app-demo`

Expected result:

    The corresponding container log files or Pod log directory for nginx-demo should be found.

---  

## Section Sixteen: Acceptance Checklist

After completing this chapter, the following should be confirmed:

    [ ] The logging Namespace has been created.
    [ ] The monitoring Namespace has been created.
    [ ] The app-demo Namespace has been created.
    [ ] The minio Namespace has been created.
    [ ] `kubectl` can access the cluster normally.
    [ ] Helm has been installed.
    [ ] The Grafana Helm repository has been added.
    [ ] Charts for grafana/loki and grafana/alloy can be searched.
    [ ] The nginx-demo application has been deployed.
    [ ] The nginx-demo Service has been created.
    [ ] `curl-test` can access nginx-demo.
    [ ] `kubectl logs` can be used to view nginx-demo logs.
    [ ] The `/var/logPrometheus is used for collecting metrics.  
Loki is responsible for storing and querying logs.  
Grafana displays both metrics and logs.  
Alloy collects and forwards logs.  
AlertManager handles alarm notifications.  

Pod logs in Kubernetes typically come from:  

    Application stdout / stderr  
      ↓  
    containerd writes to node log files  
      ↓  
    /var/log/containers  
      ↓  
    Alloy, Fluent Bit, or Filebeat  
      ↓  
    Loki, Elasticsearch, or OpenSearch  

This learning pathway for Loki follows these steps:  

    First, set up a standalone configuration.  
    Then, integrate Alloy.  
    Next, add MinIO.  
    After that, learn how to use LogQL.  
    Create Grafana dashboards.  
    Set up log alerts.  
    Finally, apply these concepts to production environments and troubleshoot issues.  

The goal of learning Loki is not just to know how to view logs but to develop a comprehensive set of skills, including:  

    Pod log collection  
    Tag management  
    LogQL queries  
    Integration with Grafana  
    Log alerts  
    Production-level management  
    Fault diagnosis  

In the next article, we will focus on:  

    02 - Practical Understanding of Loki’s Architecture and Component Responsibilities  

We will closely examine Loki’s components, service interfaces, Pod structure, and the processes involved in log writing and querying.  

---

## References  

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/  

- Installing Grafana Loki with Helm:  
  https://grafana.com/docs/loki/latest/setup/install/helm/  

- Loki Helm Chart:  
  https://github.com/grafana/loki/tree/main/production/helm/loki  

- Grafana Alloy Documentation:  
  https://grafana.com/docs/alloy/latest/  

- Collecting Kubernetes logs and forwarding them to Loki:  
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/  

- Grafana Alloy’s loki.source.kubernetes component:  
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/  

- Promtail Agent:  
  https://grafana.com/docs/loki/latest/send-data/promtail/  

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/  

- Using `kubectl logs` in Kubernetes:  
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/  

- Helm Documentation:  
  https://helm.sh/docs/  

- Grafana Documentation:  
  https://grafana.com/docs/grafana/latest/