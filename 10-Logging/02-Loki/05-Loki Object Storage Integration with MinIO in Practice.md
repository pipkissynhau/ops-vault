# 05-Loki Object Storage Integration with MinIO in Practice

## Document Description

This article is the fifth part of the dedicated learning series on Loki, aiming to deploy MinIO in a Kubernetes environment and switch the backend storage for Loki from a local file system to MinIO object storage.

In Article 04, we completed the Helm deployment of Loki in standalone mode and verified its availability through manual push/query operations.

This article will continue to achieve the following objectives:

    Deploy MinIO
      ↓
    Create Buckets for Loki usage
      ↓
    Generate AccessKey/SecretKey pairs
      ↓
    Modify Loki's values.yaml configuration
      ↓
    Configure Loki to use MinIO as object storage
      ↓
    Write test logs
      ↓
    Query test logs
      ↓
    Verify Loki data objects in MinIO

This article addresses the following key questions:

- Why does Loki require object storage?
- What are the differences between file system storage and object storage for Loki?
- Why is file system storage suitable for learning environments but object storage recommended for production use?
- What role does MinIO play in the Loki architecture?
- Why does Loki use S3-compatible configurations to connect to MinIO?
- How to deploy MinIO in Kubernetes?
- How to create Buckets such as loki-chunks, loki-ruler, and loki-admin?
- How to configure Loki to utilize MinIO?
- How to confirm that Loki has successfully written log data to MinIO?
- How to troubleshoot common issues when integrating Loki with MinIO?
- What production considerations should be taken into account when using MinIO as Loki object storage?

This article does not cover the collection of Kubernetes Pod logs using Alloy. The process of collecting Kubernetes Pod logs with Alloy will be discussed in the next article:

    06-Grafana-Alloy for Collecting K8S-Pod Logs in Practice

---

## Tags

#Loki #MinIO #S3 #ObjectStorage #Kubernetes #Helm #Grafana #LogSystem #TSDB #ObjectStorage #SRE #Observability #LogStorage

---

## Recommended Reading Path

Recommended reading path:

    10-Logs/02-Loki/05-Loki Object Storage Integration with MinIO in Practice.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand why Loki requires object storage.
    2. Comprehend the relationship between MinIO and S3-compatible APIs.
    3. Deploy MinIO in Kubernetes.
    4. Create Buckets for Loki usage.
    5. Generate access credentials for Loki.
    6. Modify Loki's Helm values to use MinIO.
    7. Update Loki using Helm upgrade.
    8. Verify that Loki’s /ready and /metrics services are functioning correctly.
    9. Manually write test logs.
    10. Query test logs using LogQL.
    11. Confirm the existence of Loki data objects in the MinIO Buckets.
    12. Master common troubleshooting methods for integrating Loki with MinIO.
    13. Understand key considerations for using MinIO/S3 storage in production environments.

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
    minio
    app-demo
    monitoring

This article primarily uses the following namespaces:

    logging:
        Where Loki is located.

    minio:
        Where MinIO is located.

### 2.2 Tools Required

You need the following tools:

    kubectl
    helm
    curl
    jq
    mc

Note:

    mc is a MinIO client that can be installed locally or run in a temporary container. This article prefers using a temporary mc container to minimize local dependencies.

### 2.3 Recommended Component Versions

Loki:

    Use the Loki Helm Release already installed in Article 04.

MinIO:

    It is recommended to use a fixed version instead of the latest release.

Example images:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

For domestic networks, you can use fixed version images synchronized to internal or cloud image repositories.

Example:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-2        minioadmin123

Do not use such a weak password in a production environment.

Create a Secret:

    kubectl create secret generic minio-root-auth \
      -n minio \
      --from-literal=MINIO_ROOT_USER=minioadmin \
      --from-literal=MINIO_ROOT_PASSWORD=minioadmin123

View:

    kubectl get secret -n minio

### 5.3 Creating a PVC

Create a file:

    minio-pvc.yaml

Content:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: minio-data
      namespace: minio
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi

Apply:

    kubectl apply -f minio-pvc.yaml

View:

    kubectl get pvc -n minio

If the PVC is in a Pending state:

    kubectl describe pvc minio-data -n minio

Common reasons:

    No default StorageClass available.
    The StorageClass does not exist.
    Dynamic provisioner encountered an error.
    The current environment does not support dynamic PV creation.

### 5.4 Deploying MinIO

Create a file:

    minio-deployment.yaml

Content:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: minio
      namespace: minio
      labels:
        app: minio
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: minio
      template:
        metadata:
          labels:
            app: minio
        spec:
          containers:
            - name: minio
              image: registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
              imagePullPolicy: IfNotPresent
              args:
                - server
                - /data
                - --console-address
                - ":9001"
              env:
                - name: MINIO_ROOT_USER
                  valueFrom:
                    secretKeyRef:
                      name: minio-root-auth
                      key: MINIO_ROOT_USER
                - name: MINIO_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: minio-root-auth
                      key: MINIO_ROOT_PASSWORD
              ports:
                - name: api
                  containerPort: 9000
                - name: console
                  containerPort: 9001
              volumeMounts:
                - name: data
                  mountPath: /data
              readinessProbe:
                httpGet:
                  path: /minio/health/ready
                  port: 9000
                initialDelaySeconds: 10
                periodSeconds: 10
              livenessProbe:
                httpGet:
                  path: /minio/health/live
                  port: 9000
                initialDelaySeconds: 30
                periodSeconds: 20
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: minio-data

Apply:

    kubectl apply -f minio-deployment.yaml

View:

    kubectl get pods -n minio -o wide

    kubectl describe pod <minio-pod-name> -n minio

    kubectl logs <minio-pod-name> -n minio --tail=100

### 5.5 Creating a Service

Create a file:

    minio-service.yaml

Content:

    apiVersion: v1
    kind: Service
    metadata:
      name: minio
      namespace: minio
      labels:
        app: minio
    spec:
      type: ClusterIP
      selector:
        app: minio
      ports:
        - name: api
          port: 9000
          targetPort: 9000
        - name: console
          port: 9001
          targetPort: 9001

Apply:

    kubectl apply -f minio-service.yaml

View:

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

### 5.6 Accessing the MinIO Console

Port forwarding:

    kubectl port-forward svc/minio 9001:9001 -n minio

Access:

    http://127.0.0.1:9001

Login:

    Username:
        minioadmin

    Password:
        minioadmin123

Note:

    In a production environment, HTTPS, strong passwords, separate users, and minimal permission policies must be used.
    This document is for experimental purposes only.

---

## Section Six:http://127.0.0.1:9001

Confirm the existence of:

    loki-chunks
    loki-ruler
    loki-admin

---

## Section 7: Creating a Secret for Loki to Access MinIO

### 7.1 Creating the Secret

The Namespace where Loki resides is logging.

It is necessary to create a Secret within the logging Namespace to access MinIO.

    kubectl create secret generic loki-minio-secret \
      -n logging \
      --from-literal=MINIO_ACCESS_KEY=minioadmin \
      --from-literal=MINIO_SECRET_KEY=minioadmin123

To view it:

    kubectl get secret loki-minio-secret -n logging

### 7.2 Why Place the Secret in the logging Namespace

Kubernetes Secrets are default Namespace-level resources.

The Loki Pod runs within the logging Namespace.

Therefore, Loki can only directly reference Secrets within the logging Namespace.

If the Secret were placed in the minio Namespace, the Loki Pod would not be able to directly access it.

### 7.3 Production Considerations

In a production environment, it is not recommended that Loki use the MinIO root user directly.

A more reasonable approach is:

    1. Create a dedicated Loki user.
    2. Establish a dedicated access policy.
    3. Allow access only to Loki-related Buckets.
    4. Do not grant MinIO administrator privileges.
    5. Manage AccessKey/SecretKey through Secrets.
    6. Avoid storing the Secrets in plaintext within Git.

---

## Section 8: Modifying Loki Values to Connect to MinIO

### 8.1 Backing Up Current Values

Before upgrading Loki, back up the current values first.

    helm get values loki -n logging -a > backup-values-loki-before-minio.yaml

Also, check the current Release:

    helm list -n logging

    helm history loki -n logging

### 8.2 Creating a MinIO-specific values File

Create:

    values-loki-monolithic-minio.yaml

Explanation:

    The following configuration is based on the monolithic mode.
    Field names may vary depending on the version of the Loki Helm Chart.
    Always refer to the output of `helm show values` before making changes.

Example Configuration:

    loki:
      auth_enabled: false

      commonConfig:
        replication_factor: 1

      schemaConfig:
        configs:
          - from: "2024-04-01"
            store: tsdb
            object_store: s3
            schema: v13
            index:
              prefix: loki_index_
              period: 24h

      storage:
        type: s3
        bucketNames:
          chunks: loki-chunks
          ruler: loki-ruler
          admin: loki-admin
        s3:
          endpoint: minio.minio.svc.cluster.local:9000
          region: us-east-1
          accessKeyId: minioadmin
          secretAccessKey: minioadmin123
          s3ForcePathStyle: true
          insecure: true

      storage_config:
        aws:
          endpoint: minio.minio.svc.cluster.local:9000
          region: us-east-1
          bucketnames: loki-chunks
          access_key_id: minioadmin
          secret_access_key: minioadmin123
          s3forcepathstyle: true
          insecure: true

      ruler:
        enable_api: true
        storage:
          type: s3
          s3:
            bucketnames: loki-ruler

      limits_config:
        allow_structured_metadata: true
        volumeenabled: true

    deploymentMode: Monolithic

    monolithic:
      replicas: 1
      persistence:
        enabled: true
        size: 10Gi

    gateway:
      enabled: true

    lokiCanary:
      enabled: false

    minio:
      enabled: false

    backend:
      replicas: 0
    read:
      replicas: 0
    write:
      replicas: 0

    ingester:
      replicas: 0
    querier:
      replicas: 0
    queryFrontend:
      replicas: 0
    queryScheduler:
      replicas: 0
    distributor:
      replicas: 0
    compactor:
      replicas: 0
    indexGateway:
      replicas: 0
    bloomPlanner:
      replicas: 0
    bloomBuilder:
      replicas: 0
    bloomGateway:
      replicas: 0

### 8.3 Safer Ways to Reference Secrets

For the sake of clarity in this example, the accessKeyId and secretAccessKey were written directly.

However, this is not recommended in a production environment.

Better practices include:

> loki-minio-rendered.yaml

### 9.2 Check if rendering was successful

    grep "^kind:" loki-minio-rendered.yaml | sort | uniq -c

### 9.3 Check the MinIO endpoint

    grep -n "minio.minio.svc.cluster.local" loki-minio-rendered.yaml

### 9.4 Check bucket names

    grep -n "loki-chunks" loki-minio-rendered.yaml

    grep -n "loki-ruler" loki-minio-rendered.yaml

    grep -n "loki-admin" loki-minio-rendered.yaml

### 9.5 Check for sensitive information

If plaintext keys are included in the values, the rendered file may contain:

    minioadmin
    minioadmin123

This is understandable in a learning environment.

In a production environment, please note:

    Do not commit loki-minio-rendered.yaml to Git.
    Do not submit plaintext secrets to the knowledge base.
    Do not include production configurations verbatim in documentation.

---

## Section 10: Upgrading Loki Using MinIO

### 10.1 Execute Helm upgrade

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio.yaml

### 10.2 Check Helm status

    helm list -n logging

    helm status loki -n logging

    helm history loki -n logging

### 10.3 Check Pod changes

    kubectl get pods -n logging -o wide

If the Pod is being rebuilt, wait until it is in the Running state:

    kubectl rollout status statefulset/<loki-statefulset-name> -n logging

If Loki is a Deployment:

    kubectl rollout status deploy/<loki-deployment-name> -n logging

The actual resource names can be found by running:

    kubectl get deploy,statefulset -n logging

### 10.4 Check Loki logs

    kubectl logs <loki-pod-name> -n logging --tail=200

To filter key information:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "s3|minio|storage|bucket|error|warn|failed|flush|chunk"

Pay special attention to:

    Whether there are errors connecting to MinIO.
    Whether the bucket does not exist.
    Whether there are issues with access keys or secret keys.
    Whether there are errors with endpoints.
    Whether there are permission issues.

---

## Section 11: Verifying Loki API

### 11.1 Port forwarding

It is recommended to use a Gateway:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

If there is no gateway, forward the Loki Service directly:

    kubectl port-forward svc/loki 3100:3100 -n logging

### 11.2 Verify readiness

    curl -s http://127.0.0.1:3100/ready

Expected response:

    ready

### 11.3 Verify metrics

    curl -s http://127.0.0.1:3100/metrics | head

To filter Loki-specific metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

### 11.4 Verify labels

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

If the upgrade was just completed and no new logs have been written, there may be few labels available.

---

## Section 12: Writing Test Logs

### 12.1 Prepare a timestamp

    TS=$(date +%s%N)

Display the timestamp:

    echo $TS

### 12.2 Write test logs

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-minio-test\",
              \"namespace\": \"app-demo\"",
              \"app\": \"loki-minio-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki with minio object storage\"]
            ]
          }
        ]
      }"

If no output is displayed, it usually means the write request was successful.

### 12.3 Query test logs

    curl -G -s "http://```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-minio-test"}' \
      --data-urlencode 'limit=20' | jq
```

After waiting, check MinIO:

```bash
mc find local/loki-chunks
```

---

## Section Fourteen: Verification via the MinIO Console

### 14.1 Port Forwarding for the Console

Use the following command to forward the port:

```bash
kubectl port-forward svc/minio 9001:9001 -n minio
```

Access it at:

```bash
http://127.0.0.1:9001
```

Log in with either:

```bash
minioadmin
```
or

```bash
minioadmin123
```

### 14.2 Viewing Buckets

Check the following buckets:

```bash
loki-chunks
loki-ruler
loki-admin
```

Pay special attention to whether any data objects written by Loki are present in the `loki-chunks` bucket.

**Note:** The object paths may not be direct log file names. Loki organizes chunks and indexes according to its internal structure. You should not expect to see file names like `app.log`.

---

## Section Fifteen: Troubleshooting After Integrating Loki with MinIO

### 15.1 Loki Pod CrashLoopBackOff

Check the following:

```bash
kubectl get pods -n logging
kubectl describe pod <loki-pod-name> -n logging
kubectl logs <loki-pod-name> -n logging --previous --tail=200
```

Filter for relevant errors using:

```bash
kubectl logs <loki-pod-name> -n logging --previous --tail=500 | grep -Ei "s3|minio|bucket|storage|error|failed|access|secret|permission"
```

**Common causes:** Mismatch in `values` fields, incorrect `storage_config` settings, `schemaConfig` object_store not set to `s3`, non-existent bucket, incorrect endpoint, invalid access keys or secret keys, MinIO service unreachable, MinIO using HTTP while Loki is configured for HTTPS, `s3forcepathstyle` not enabled, mismatch between `replication_factor` and the actual number of replicas.

### 15.2 `/ready` Status Not Being "Ready"

Check:

```bash
curl -s http://127.0.0.1:3100/ready
```

If it's not ready, check the logs for more details:

```bash
kubectl logs <loki-pod-name> -n logging --tail=200
```

**Possible causes:** Loki not fully started, storage initialization failed, MinIO unreachable, ring not ready, configuration errors, Gateway backend unavailable.

### 15.3 `push` Operation Returns 500 Error

Possible reasons include internal writing failures in Loki or MinIO, insufficient bucket permissions, non-existent bucket, incorrect endpoint, or anomalies with the Loki Ingester.

**Troubleshooting steps:** Check the logs for details:

```bash
kubectl logs <loki-pod-name> -n logging --tail=300
kubectl get pods -n minio
kubectl logs <minio-pod-name> -n minio --tail=200
```

### 15.4 `push` Operation Returns 400 Error

Possible causes include incorrect JSON format, timestamp format issues, errors in the `values` fields or label formats, or timestamps being too old or too new.

**Check steps:** Verify the `TS` value and the date format. Ensure the curl request body conforms to the Loki push API format.

### 15.5 Data Not Found When Querying

First, check the labels:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq
```

Then, check the `job` field:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/label/job/values | jq
```

Finally, try querying within a specific range:

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-minio-test"}' \
      --data-urlencode 'limit=10' | jq
```

**Possible causes:** Writing failures, incorrect query tags, incorrect time range specified, data not yet flushed in Loki but still available, wrong### Helm History for Loki - Logging

**Rollback:**

```bash
helm rollback loki <REVISION> -n logging
```

### 17.2 Confirmation Before Rolling Back

If you need to roll back after switching from the filesystem to MinIO, you must confirm the following:

- Whether the original PVC data is still present.
- Whether the data newly written to MinIO needs to be retained.
- Whether it is acceptable to discard test data.
- Whether data migration is required.
- Whether Grafana queries will be affected.

### 17.3 Backing Up Current Values

After upgrading, save the current values:

```bash
helm get values loki -n logging -a > backup-values-loki-after-minio.yaml
```

It is recommended to keep the following files:

- `backup-values-loki-before-minio.yaml`
- `backup-values-loki-after-minio.yaml`
- `values-loki-monolithic-minio.yaml`

---

## Section 18: Practical Tasks

### 18.1 Task 1: Deploy MinIO

**Execution:**

```bash
kubectl create namespace minio
kubectl create secret generic minio-root-auth \
  -n minio \
  --from-literal=MINIO_ROOT_USER=minioadmin \
  --from-literal=MINIO_ROOT_PASSWORD=minioadmin123
kubectl apply -f minio-pvc.yaml
kubectl apply -f minio-deployment.yaml
kubectl apply -f minio-service.yaml
```

**Acceptance:**

- [ ] The MinIO Pod is running.
- [ ] The MinIO Service exists.
- [ ] The MinIO endpoint is available.
- [ ] The MinIO Console can be accessed.

### 18.2 Task 2: Create a Bucket

**Execution:**

```bash
kubectl run minio-mc \
  --rm -it \
  --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  -n minio \
  -- sh
```

**Inside the container:**

```bash
mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123
mc mb local/loki-chunks
mc mb local/loki-ruler
mc mb local/loki-admin
mc ls local
```

**Acceptance:**

- [ ] The `loki-chunks` bucket was created successfully.
- [ ] The `loki-ruler` bucket was created successfully.
- [ ] The `loki-admin` bucket was created successfully.

### 18.3 Task 3: Configure Loki to Use MinIO

**Execution:**

```bash
helm get values loki -n logging -a > backup-values-loki-before-minio.yaml
helm template loki grafana-community/loki \
  --namespace logging \
  --version <CHART_VERSION> \
  -f values-loki-monolithic-minio.yaml \
  > loki-minio-rendered.yaml
helm upgrade loki grafana-community/loki \
  --namespace logging \
  --version <CHART_VERSION> \
  -f values-loki-monolithic-minio.yaml
```

**Acceptance:**

- [ ] The Helm template execution was error-free.
- [ ] The Helm upgrade was successful.
- [ ] The Loki Pod is running.
- [ ] The `/ready` endpoint indicates that Loki is ready for use.

### 18.4 Task 4: Write and Query Logs

**Execution:**

```bash
kubectl port-forward svc/loki-gateway 3100:80 -n logging
```

In another terminal:

```bash
TS=$(date +%s%N)
curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
  -H "Content-Type: application/json" \
  -d "{
    \"streams\": [
      {
        \"stream\": {
          \"job\": \"manual-minio-test\"",
          \"namespace\": \"app-demo\",
          \"app\": \"loki-minio-test\"
        },
        \"values\": [
          [\"${TS}\", \"hello loki with minio object storage\"]
        ]
      }
    ]
  }"

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="manual-minio-test"}' \
  --data-urlencode 'limit=10' | jq
```

**Acceptance:**

- [ ] The log was successfully written.
- [ ] The query returned the expected result.
- [ ]### 20.4 Myth Four: The MinIO root user can be used for Loki

Not recommended.

For production use, it is advisable to create a dedicated user and policies for Loki, allowing access only to Loki-related Buckets.

### 20.5 Myth Five: S3 configurations are only applicable to AWS

Incorrect.

MinIO is compatible with the S3 API, so when integrating Loki with MinIO, S3-related configurations can still be used.

### 20.6 Myth Six: Object storage solves all Loki issues

Wrong.

Object storage primarily addresses long-term storage and scalability concerns.

Other factors that need attention include:

    Number of tags
    Query scope
    Write throttling
    Log volume management
    Loki's own monitoring
    The Compactor
    Ruler
    Gateway
    Stability of Alloy data collection

---

## Section 21: Advanced Understanding for Production Environments

### 21.1 Recommended Production Stack

For production, it is recommended to use the following stack:

    Grafana Alloy
      ↓
    Loki Gateway
      ↓
    Loki writing components
      ↓
    Object storage
      ↓
    Grafana queries
      ↓
    Loki Ruler alerts
      ↓
    AlertManager notifications

Object storage options include:

    Enterprise MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    Other S3-compatible storage solutions

### 21.2 Monitoring MinIO in Production

MinIO itself also requires monitoring:

    Whether the MinIO Pod is running
    Availability of the MinIO API
    Disk capacity
    Disk I/O performance
    Bucket capacity
    Number of S3 requests
    S3 4xx/5xx errors
    Request latency
    Node health status
    Erasure code status
    Certificate validity period

### 21.3 Impact of Failures in Loki + MinIO

If MinIO becomes unavailable, it may affect:

    Chunk flushing in Loki
    Access to historical logs
    Stability of new log writes
    Operation of the Compactor
    Storage of Ruler data
    Query latency

Therefore, object storage in a production environment should not be a single point of failure.

### 21.4 Retention Periods

The retention period for Loki logs needs to take into account:

    Loki's own retention settings
    The behavior of the Compactor
    The lifecycle of the object storage
    Compliance requirements
    Cost considerations

It is important not to prematurely delete data that Loki still requires. It is essential to understand Loki's retention configuration and the Compactor's behavior.

---

## Section 22: Summary

This article has covered the practical implementation of integrating Loki with MinIO for object storage.

Key components include:

    MinIO deployment
    MinIO service
    loki-chunks, loki-ruler, and loki-admin Buckets
    Configuration of Loki values using S3/MinIO
    Helm updates
    Final verification of the setup

Key takeaways are:

    While file systems can be used for Loki, object storage is more recommended for production.
    MinIO integrates with Loki through an S3-compatible API.
    Log writes to Loki may not be immediately available in object storage.
    Recent logs might need to be queried from the Ingester.
    Object store data may only become accessible after chunk flushing.
    In a production environment, considerations such as high availability, security, permissions, HTTPS, and capacity management are crucial.

With this setup, the Loki server now has a production-ready storage foundation.

The next topic will focus on:

    06-Grafana-Alloy for Collecting Kubernetes Pod Logs

The main objective is to ensure that Kubernetes Pod stdout/stderr logs can be collected by Alloy, sent to Loki, and then queried in Grafana/LogQL based on namespace, pod, and container details.

---

## References

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Configuring Storage:
  https://grafana.com/docs/loki/latest/setup/install/helm/configure-storage/

- Loki Storage:
  https://grafana.com/docs/loki/latest/configure/storage/

- Loki Configuration Examples:
  https://grafana.com/docs/loki/latest/configure/examples/configuration-examples/

- Loki Helm Chart:
  https://github.com/grafana/loki/tree/main/production/helm/loki

- Installing Grafana Loki with Helm:
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Installing the Monolithic Helm Chart:
  https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/

- MinIO Documentation:
  https://min.io/docs/minio/kubernetes/upstream/

- MinIO Client:
  https://min.io/docs/minio/linux/reference/minio-mc.html

- Kubernetes Persistent Volumes:
  https