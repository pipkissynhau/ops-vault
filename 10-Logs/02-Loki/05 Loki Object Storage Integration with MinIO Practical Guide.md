# 05-Loki Object Storage Access to MinIO Practical Guide

## Document Overview

This is the fifth article in the Loki specialized learning series, used to deploy MinIO in a Kubernetes environment and switch Loki's single-node mode backend storage from local file system to MinIO object storage.

The fourth article has already completed Loki single-node mode Helm deployment and verified Loki service availability through manual push/query.

This article continues to achieve the following goals:

    Deploy MinIO
      ↓
    Create Buckets for Loki usage
      ↓
    Create AccessKey / SecretKey
      ↓
    Modify Loki values.yaml
      ↓
    Loki uses MinIO as object storage
      ↓
    Write test logs
      ↓
    Query test logs
      ↓
    Verify Loki data objects in MinIO

This article focuses on answering the following questions:

- Why does Loki need object storage;
- What are the differences between Loki's file system storage and object storage;
- Why can learning environments use file systems first but production recommends object storage;
- What role does MinIO play in Loki architecture;
- Why does Loki access MinIO through S3 compatible configuration;
- How to deploy MinIO in Kubernetes;
- How to create Buckets like loki-chunks, loki-ruler, loki-admin;
- How to configure Loki to use MinIO;
- How to verify Loki has written log data to MinIO;
- How to troubleshoot common errors after Loki accesses MinIO;
- What production considerations are needed when using MinIO as Loki's object storage.

This article does not cover Alloy log collection for Kubernetes Pods.

Alloy log collection for Kubernetes Pods will be covered in the next article:

    06-Grafana-AlloyCollectionK8S-PodLogic Action

---

## Tags

#Loki #MinIO #S3 #ObjectStorage #Kubernetes #Helm #Grafana #LogSystem #TSDB #ObjectStorage #SRE #Observation #LogStorage

---

## Recommended Path

Recommended path:

    10-Log/02-Loki/05-LokiObject Storage AccessMinIOActual.md

---

## One: Experiment Objectives

After completing this article, you should be able to:

    1. Understand why Loki needs object storage.
    2. Understand the relationship between MinIO and S3 compatible API.
    3. Deploy MinIO in Kubernetes.
    4. Create Buckets for Loki usage.
    5. Create access credentials for Loki usage.
    6. Modify Loki Helm values to make Loki use MinIO.
    7. Update Loki through Helm upgrade.
    8. Verify Loki /ready, /metrics are normal.
    9. Manually write test logs.
    10. Query test logs through LogQL.
    11. Confirm Loki data objects in MinIO Bucket.
    12. Master common troubleshooting methods after Loki accesses MinIO.
    13. Understand key considerations for MinIO/S3 storage in production environments.

---

## Two: Experiment Environment

### 2.1 Kubernetes Cluster

Experiment nodes:

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

This article focuses on:

    logging:
        Namespace where Loki resides.

    minio:
        Namespace where MinIO resides.

### 2.2 Tools

Required:

    kubectl
    helm
    curl
    jq
    mc

Notes:

    mc is MinIO Client.
    It can be installed locally or run in a temporary container.
    This article prefers using temporary mc container to reduce local dependencies.

### 2.3 Component Version Recommendations

Loki:

    Use the Loki Helm Release installed in the fourth article.

MinIO:

    Recommend fixed version, do not use latest.

Example images:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

Domestic network environment can use fixed version images synchronized to internal or cloud image repositories.

Example:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Notes:

    Image versions must be fixed.
    Do not recommend using latest.
    Production version selection needs to consider security patches, licenses, enterprise compliance, and upgrade strategies.

---

## Three: Why Loki Needs Object Storage

### 3.1 What Data Does Loki Store

Loki mainly stores:

    Log chunks
    Index data
    Ruler rule data
    Admin related data

Under TSDB schema, Loki organizes log data into chunks by time and log stream, and writes data to backend storage.

### 3.2 Issues with File System Storage

In the fourth article, file system storage was used for quick learning and verification.

This approach is suitable for:

    Learning
    Temporary verification
    Small-scale experiments
    Local testing

But it's unsuitable for long-term production use.

Reasons:

    Pod restarts may affect data.
    Node failures may affect data.
    Local disk capacity is limited.
    Expansion is inconvenient.
    Multi-replica sharing is difficult.
    Backup and recovery is complex.
    High availability capabilities are insufficient.

### 3.3 Advantages of Object Storage

Object storage is suitable for:

    Storing large amounts of log chunks
    Supporting long-term retention
    Supporting capacity expansion
    Supporting multi-replica Loki access
    Reducing local disk dependency
    Supporting more production-ready deployment modes

Common object storages:

    MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    Google Cloud Storage
    Azure Blob

This article uses: /think

MinIO

Reasons:

    MinIO supports S3-compatible API.
    Suitable for local experimentation.
    Suitable for Kubernetes deployment.
    Easy to observe objects written to Loki.
    Can be migrated to cloud vendor object storage later.

---

## FourI don't know.Relationship Between Loki and MinIO

### 4.1 Position of MinIO in the Architecture

After integrating MinIO, the chain becomes:

    Alloy / manual push
      ↓
    Loki Gateway
      ↓
    Distributor
      ↓
    Ingester
      ↓
    Chunk / Index
      ↓
    MinIO Bucket

Query chain:

    Grafana / curl query_range
      ↓
    Loki Query API
      ↓
    Querier
      ↓
    Ingester's recent data
      +
    MinIO's historical data
      ↓
    Return log results

### 4.2 Why Loki Uses S3 Configuration to Integrate with MinIO

MinIO implements S3-compatible API.

Therefore, when configuring MinIO, Loki typically uses S3-type configuration.

Key parameters:

    endpoint
    access_key_id
    secret_access_key
    bucketnames
    s3forcepathstyle
    insecure

Among them:

    endpoint:
        MinIO service address.

    access_key_id:
        MinIO username or AccessKey.

    secret_access_key:
        MinIO password or SecretKey.

    bucketnames:
        Bucket used by Loki.

    s3forcepathstyle:
        MinIO usually requires enabling path-style.

    insecure:
        Set to true when using HTTP.

### 4.3 Bucket Planning

In a learning environment, you can use the default bucket names:

    chunks
    ruler
    admin

You can also use clearer names:

    loki-chunks
    loki-ruler
    loki-admin

This article recommends using clear names:

    loki-chunks
    loki-ruler
    loki-admin

Explanation:

    chunks:
        Stores log data chunks.

    ruler:
        Stores data related to Loki Ruler rules.

    admin:
        Stores admin-related data, used in some scenarios.

Notes:

    If it's a public cloud S3, bucket names usually need to be globally unique.
    If it's local MinIO, bucket names only need to be unique within the current MinIO.

---

## FiveI don't know.Deploying MinIO

### 5.1 Create Namespace

    kubectl create namespace minio

Verification:

    kubectl get ns minio

### 5.2 Create MinIO Secret

Learning environment example account:

    root user:
        minioadmin

    root password:
        minioadmin123

Do not use such weak passwords in production environments.

Create Secret:

    kubectl create secret generic minio-root-auth \
      -n minio \
      --from-literal=MINIO_ROOT_USER=minioadmin \
      --from-literal=MINIO_ROOT_PASSWORD=minioadmin123

View:

    kubectl get secret -n minio

### 5.3 Create PVC

Create file:

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

If PVC is Pending:

    kubectl describe pvc minio-data -n minio

Common reasons:

    No default StorageClass.
    StorageClass does not exist.
    Dynamic provisioner is abnormal.
    Current environment does not support dynamic PV creation.

### 5.4 Deploy MinIO

Create file:

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
                  name: min
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

Check:
    kubectl get pods -n minio -o wide

    kubectl describe pod <minio-pod-name> -n minio

    kubectl logs <minio-pod-name> -n minio --tail=100

### 5.5 Create Service

Create file:
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

Check:
    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

### 5.6 Access MinIO Console

Port forward:
    kubectl port-forward svc/minio 9001:9001 -n minio

Access:
    http://127.0.0.1:9001

Login:
    Username:
        minioadmin

    Password:
        minioadmin123

Note:
    Production environments must use HTTPS, strong passwords, dedicated users, and minimal privilege policies.
    This document is only for experimental learning purposes.

---

## SixI don't know.Create Loki Bucket

### 6.1 Use mc Temporary Container

Run mc temporary Pod:
    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh /think

Enter the container and set alias:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

Check:

    mc admin info local

### 6.2 Create Bucket

Execute inside the mc container:

    mc mb local/loki-chunks

    mc mb local/loki-ruler

    mc mb local/loki-admin

If prompted that the bucket already exists, you can ignore the message.

Check:

    mc ls local

Expected output:

    loki-admin
    loki-chunks
    loki-ruler

### 6.3 Test Object Writing

    echo "hello minio for loki" > /tmp/test.txt

    mc cp /tmp/test.txt local/loki-chunks/test.txt

Check:

    mc ls local/loki-chunks

Delete the test object:

    mc rm local/loki-chunks/test.txt

Exit:

    exit

### 6.4 Verify via Console

Open MinIO Console:

    http://127.0.0.1:9001

Confirm existence of:

    loki-chunks
    loki-ruler
    loki-admin

---

## SevenI don't know.Create Loki Access Secret for MinIO

### 7.1 Create Secret

The namespace where Loki resides is logging.

You need to create a secret for accessing MinIO in the logging namespace.

    kubectl create secret generic loki-minio-secret \
      -n logging \
      --from-literal=MINIO_ACCESS_KEY=minioadmin \
      --from-literal=MINIO_SECRET_KEY=minioadmin123

Check:

    kubectl get secret loki-minio-secret -n logging

### 7.2 Why Secret Must Be in logging Namespace

Kubernetes Secrets are namespace-scoped by default.

Loki Pods run in the logging namespace.

Therefore Loki can only directly reference secrets in the logging namespace.

If the secret is placed in the minio namespace, Loki Pods cannot directly reference it.

### 7.3 Production Notes

In production environments, it's not recommended for Loki to directly use the MinIO root user.

A more reasonable approach:

    1. Create a dedicated Loki user.
    2. Create a dedicated access policy.
    3. Only allow access to Loki-related buckets.
    4. Do not grant MinIO administrator permissions.
    5. AccessKey/SecretKey should be managed via Secret.
    6. Secrets should not be committed to Git in plaintext.

---

## EightI don't know.Modify Loki values to Integrate with MinIO

### 8.1 Backup Current values

Before upgrading Loki, back up the current values.

    helm get values loki -n logging -a > backup-values-loki-before-minio.yaml

Also check the current release:

    helm list -n logging

    helm history loki -n logging

### 8.2 Create MinIO Version values File

Create:

    values-loki-monolithic-minio.yaml

Notes:

    The following configuration is an example in monolithic mode.
    Field names may vary across different Loki Helm Chart versions.
    You must first compare with the output of helm show values during actual operation.

Example configuration:

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
  volume_enabled: true

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

### 8.3 More Secure Secret Reference Methods

Above, for learning clarity, we directly wrote accessKeyId and secretAccessKey.

This is not recommended for production environments.

More recommended approaches:

  Use existingSecret
  Or inject via environment variables
  Or inject via external Secret management systems

However, different Helm Chart versions may support different Secret reference fields.

Therefore, practical steps:

  1. helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml
  2. Search for existingSecret / secret / s3 / storage
  3. Configure according to current Chart support
  4. Avoid committing plaintext keys to Git

Search:

  grep -n "existingSecret" values-loki-default.yaml

  grep -n "accessKey" values-loki-default.yaml

  grep -n "secretAccessKey" values-loki-default.yaml

  grep -n "storage:" values-loki-default.yaml

### 8.4 endpoint Writing Instructions

Accessing MinIO within a Kubernetes cluster:

  minio.minio.svc.cluster.local:9000

Meaning:

  Service name:
      minio

  Namespace:
      minio

  Port:
      9000

If Loki and MinIO are in the same Namespace, it can be abbreviated as:

  minio:9000

In this article, Loki is in logging, MinIO is in minio, so it is recommended to use the full FQDN.

### 8.5 insecure Explanation

This article uses HTTP for MinIO, therefore:

  insecure: true

In production environments, if using HTTPS, do not set this to true.

Production recommendations:

  MinIO API uses HTTPS
  Loki configures TLS
  Certificate is trusted
  Do not skip validation

---

## Nine. Helm Rendering Check

### 9.1 Execute helm template

  helm template loki grafana-community/loki \
    --namespace logging \
    --version <CHART_VERSION> \
    -f values-loki-monolithic-minio.yaml \
    > loki-minio-rendered.yaml

### 9.2 Check if rendering was successful

  grep "^kind:" loki-minio-rendered.yaml | sort | uniq -c

### 9.3 Check MinIO endpoint

  grep -n "minio.minio.svc.cluster.local" loki-minio-rendered.yaml

### 9.4 Check bucket names

  grep -n "loki-chunks" loki-minio-rendered.yaml

  grep -n "loki-ruler" loki-minio-rendered.yaml

  grep -n "loki-admin" loki-minio-rendered.yaml

### 9.5 Check sensitive information

If plaintext keys were written in values, the rendered file may contain:

  minioadmin
  minioadmin123

This can be understood in a learning environment.

In production environments, note:

  Do not commit loki-minio-rendered.yaml to Git.
  Do not commit plaintext Secrets to the knowledge base.
  Do not directly place production configurations in documentation.

---

## Ten. Upgrading Loki with MinIO

### 10.1 Execute helm upgrade

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

If Pods are being rebuilt, wait for Running: /think

kubectl rollout status statefulset/<loki-statefulset-name> -n logging

If Loki is a Deployment:

    kubectl rollout status deploy/<loki-deployment-name> -n logging

The actual resource name is determined by:

    kubectl get deploy,statefulset -n logging

---

### 10.4 View Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=200

Filter critical information:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "s3|minio|storage|bucket|error|warn|failed|flush|chunk"

Focus on:

    Whether connecting to MinIO failed
    Whether the bucket does not exist
    Whether the access key is incorrect
    Whether the secret key is incorrect
    Whether the endpoint is incorrect
    Whether there is insufficient permission

---

## Eleven. Verify Loki API

### 11.1 Port Forwarding

Recommended via Gateway:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

If there is no gateway, forward Loki Service:

    kubectl port-forward svc/loki 3100:3100 -n logging

### 11.2 Verify Ready

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

### 11.3 Verify Metrics

    curl -s http://127.0.0.1:3100/metrics | head

Filter Loki metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

### 11.4 Verify Labels

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

If the upgrade is just completed and no new logs have been written, labels may be limited.

---

## Twelve. Write Test Logs

### 12.1 Prepare Timestamp

    TS=$(date +%s%N)

View:

    echo $TS

### 12.2 Write Test Logs

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-minio-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"loki-minio-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki with minio object storage\"]
            ]
          }
        ]
      }"

If there is no output, it typically indicates the write request was successful.

### 12.3 Query Test Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-minio-test"}' \
      --data-urlencode 'limit=10' | jq

Expected to see:

    hello loki with minio object storage

### 12.4 Query Labels

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/job/values" | jq

Expected to include:

    manual-minio-test

---

## Thirteen. Verify if Loki Data Exists in MinIO

### 13.1 Enter mc Temporary Container

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Set alias:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

### 13.2 View Bucket

    mc ls local

View chunks:

    mc ls local/loki-chunks

Recursive view:

    mc find local/loki-chunks

View ruler:

    mc ls local/loki-ruler

Exit:

    exit

### 13.3 Why Objects Might Not Be Visible Immediately After Writing

Loki does not necessarily write objects to MinIO immediately after each log entry.

Reasons:

    Ingester organizes logs in memory and WAL first.
    Logs are flushed in chunks according to chunking policies.
    Data writing to object storage may have delays.
    Querying recent data may directly come from Ingester.

Therefore:

    Push success + query success:
        Indicates the Loki write and query pipeline is normal.

    MinIO cannot immediately see objects:
        Does not necessarily indicate failure.

You can write more logs or wait for some time before checking again.

### 13.4 Quickly Generate More Test Logs

Execute multiple writes:

```bash
for i in $(seq 1 100); do
  TS=$(date +%s%N)
  curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
    -H "Content-Type: application/json" \
    -d "{
      \"streams\": [
        {
          \"stream\": {
            \"job\": \"manual-minio-test\",
            \"namespace\": \"app-demo\",
            \"app\": \"loki-minio-test\"
          },
          \"values\": [
            [\"${TS}\", \"hello loki with minio object storage line ${i}\"]
          ]
        }
      ]
    }" >/dev/null
  sleep 1
done
```

Query:

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="manual-minio-test"}' \
  --data-urlencode 'limit=20' | jq
```

Wait and check MinIO:

```bash
mc find local/loki-chunks
```

---

## FourteenI don't know.Verification via MinIO Console

### 14.1 Port Forwarding Console

```bash
kubectl port-forward svc/minio 9001:9001 -n minio
```

Access:

```
http://127.0.0.1:9001
```

Login:

```
minioadmin
minioadmin123
```

### 14.2 View Bucket

Check:

```
loki-chunks
loki-ruler
loki-admin
```

Focus on:

```
Whether Loki-written data objects appear in loki-chunks.
```

Note:

```
Object paths may not be intuitive log filenames.
Loki organizes chunks and indexes internally.
Do not expect to see filenames like app.log.
```

---

## FifteenI don't know.Troubleshooting After Loki Integrates with MinIO

### 15.1 Loki Pod CrashLoopBackOff

Check:

```bash
kubectl get pods -n logging

kubectl describe pod <loki-pod-name> -n logging

kubectl logs <loki-pod-name> -n logging --previous --tail=200
```

Focus filtering:

```bash
kubectl logs <loki-pod-name> -n logging --previous --tail=500 | grep -Ei "s3|minio|bucket|storage|error|failed|access|secret|permission"
```

Common causes:

```
Values field mismatch.
Storage_config formatting error.
schemaConfig object_store not changed to s3.
Bucket does not exist.
Endpoint error.
Access key error.
Secret key error.
MinIO Service unreachable.
MinIO uses HTTP, but Loki is configured for HTTPS.
s3forcepathstyle not enabled.
replication_factor mismatch with replica count.
```

### 15.2 /ready Not Ready

Check:

```bash
curl -s http://127.0.0.1:3100/ready
```

If not ready:

```bash
kubectl logs <loki-pod-name> -n logging --tail=200
```

Possible causes:

```
Loki startup not completed.
Storage initialization failed.
MinIO unreachable.
ring not ready.
Configuration error.
Gateway backend unavailable.
```

### 15.3 push Returns 500

Possible causes:

```
Loki internal write failure.
MinIO write failure.
Bucket permission insufficient.
Bucket does not exist.
Endpoint error.
Loki Ingester anomaly.
```

Troubleshoot:

```bash
kubectl logs <loki-pod-name> -n logging --tail=300

kubectl get pods -n minio

kubectl logs <minio-pod-name> -n minio --tail=200
```

### 15.4 push Returns 400

Possible causes:

```
JSON format error.
Timestamp format error.
Values field error.
Label format error.
Timestamp too old or too new.
```

Check:

```bash
echo $TS

date

Check if the curl request body conforms to Loki push API format.
```

### 15.5 query Not Finding Data

First check labels:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq
```

Then check job:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/label/job/values | jq
```

Then check query_range:

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="manual-minio-test"}' \
  --data-urlencode 'limit=10' | jq
```

Possible causes:

```
Write failure.
Query label error.
Query time range mismatch.
Loki has not flushed yet, but recent data should still be queryable.
Port forwarding to wrong Service.
Gateway forwarding anomaly.
```

### 15.6 MinIO Bucket Empty

If Loki can write and query but the bucket is temporarily empty, it may not be an issue.

Possible causes:

Data is still in the Ingester's memory.  
Chunk has not been flushed.  
Write volume is too low.  
Wait time is too short.  
Checked the error Bucket.  
Loki is still using filesystem in practice.  
Values configuration has not taken effect.  

Check Loki's current configuration:  

    helm get values loki -n logging -a  

Check rendered result:  

    helm get manifest loki -n logging | grep -Ei "s3|minio|bucket|filesystem|object_store"  

Check Loki logs:  

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "s3|minio|bucket|filesystem|storage"  

### 15.7 MinIO Service is unreachable  

Run a temporary Pod from the logging Namespace for testing:  

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh  

Inside the container:  

    curl -I http://minio.minio.svc.cluster.local:9000  

Exit:  

    exit  

If it's unreachable:  

    kubectl get svc -n minio  

    kubectl get endpoints minio -n minio  

    kubectl get pods -n minio -o wide  

    kubectl logs <minio-pod-name> -n minio --tail=200  

---  

## Sixteen, Production Considerations for Loki Using MinIO  

### 16.1 Do not use root user for Loki  

The learning environment uses:  

    minioadmin  

Production environment should create:  

    Loki dedicated user  
    Loki dedicated AccessKey  
    Policy allowing access only to Loki-related Buckets  

Principle:  

    Minimal permissions  

### 16.2 Do not submit keys in plain text  

Do not submit the following to Git:  

    accessKeyId  
    secretAccessKey  
    root password  
    Secret plain text  
    YAML with keys after helm template  

Recommendations:  

    Use Kubernetes Secret  
    Use External Secrets  
    Use Vault  
    Use SealedSecrets  
    Use cloud vendor KMS / Secret Manager  

### 16.3 HTTPS must be used in production  

The learning environment uses HTTP.  

Production recommendations:  

    Enable HTTPS for MinIO API  
    Loki validates MinIO certificate  
    Do not use insecure: true  
    Certificate issued by trusted CA  
    TLS should also be considered for internal services  

### 16.4 MinIO cannot be single-node  

This is a single-replica MinIO.  

Production does not recommend:  

    Single Pod MinIO  
    Single PVC MinIO  
    No backup MinIO  

Production recommendations:  

    Distributed MinIO  
    Multi-node  
    Erasure Coding  
    Regular backups  
    Monitor capacity  
    Monitor disks  
    Monitor S3 API latency  
    Monitor 4xx / 5xx  

### 16.5 Object storage capacity planning  

Need to estimate:  

    Daily log volume  
    Size after compression  
    Retention days  
    Replication or erasure coding overhead  
    Query access pattern  
    Growth rate  

Rough formula:  

    Total capacity = Daily write volume × Retention days × Redundancy factor  

Also reserve:  

    20% - 30% safety margin  

### 16.6 Bucket Lifecycle Policy  

In production, you can combine:  

    Loki retention  
    MinIO lifecycle  
    Object storage lifecycle policy  

Note:  

    Do not let MinIO lifecycle and Loki retention conflict.  
    Do not allow object storage to delete data Loki still needs.  
    Prioritize understanding Loki retention and compactor behavior.  

---  

## Seventeen, Backup and Rollback  

### 17.1 Helm Rollback  

Check history:  

    helm history loki -n logging  

Rollback:  

    helm rollback loki <REVISION> -n logging  

### 17.2 Confirm Before Rollback  

If switching from filesystem to MinIO and need to rollback, confirm:  

    Whether the original PVC data still exists.  
    Whether new data written to MinIO needs to be retained.  
    Whether it's acceptable to discard test data.  
    Whether data migration is needed.  
    Whether Grafana queries are affected.  

### 17.3 Backup Current values  

Save after upgrade:  

    helm get values loki -n logging -a > backup-values-loki-after-minio.yaml  

Recommend to keep:  

    backup-values-loki-before-minio.yaml  
    backup-values-loki-after-minio.yaml  
    values-loki-monolithic-minio.yaml  

---  

## Eighteen, Practical Tasks  

### 18.1 Task One: Deploy MinIO  

Execute:  

    kubectl create namespace minio  

    kubectl create secret generic minio-root-auth \
      -n minio \
      --from-literal=MINIO_ROOT_USER=minioadmin \
      --from-literal=MINIO_ROOT_PASSWORD=minioadmin123  

    kubectl apply -f minio-pvc.yaml  

    kubectl apply -f minio-deployment.yaml  

    kubectl apply -f minio-service.yaml  

Verification:  

    [ ] MinIO Pod Running  
    [ ] MinIO Service exists  
    [ ] MinIO Endpoint is not empty  
    [ ] MinIO Console is accessible  

### 18.2 Task Two: Create Bucket  

Execute: /think

kubectl run minio-mc \
  --rm -it \
  --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  -n minio \
  -- sh

Container:

  mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

  mc mb local/loki-chunks

  mc mb local/loki-ruler

  mc mb local/loki-admin

  mc ls local

Verification:

  [ ] loki-chunks created successfully
  [ ] loki-ruler created successfully
  [ ] loki-admin created successfully

### 18.3 Task Three: Configure Loki to Use MinIO

Execute:

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

Verification:

  [ ] helm template no errors
  [ ] helm upgrade successful
  [ ] Loki Pod Running
  [ ] Loki /ready returns ready

### 18.4 Task Four: Write and Query Logs

Execute:

  kubectl port-forward svc/loki-gateway 3100:80 -n logging

Another terminal:

  TS=$(date +%s%N)

  curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
    -H "Content-Type: application/json" \
    -d "{
      \"streams\": [
        {
          \"stream\": {
            \"job\": \"manual-minio-test\",
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

Verification:

  [ ] Write successful
  [ ] Query successful
  [ ] Can see hello loki with minio object storage

### 18.5 Task Five: Verify MinIO Data

Execute:

  kubectl run minio-mc \
    --rm -it \
    --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    -n minio \
    -- sh

Container:

  mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

  mc find local/loki-chunks

Verification:

  [ ] Can see Loki-related objects, or understand why objects may not appear immediately
  [ ] Loki push/query are normal
  [ ] Loki logs have no MinIO write errors

---

## Nineteen, Acceptance Checklist

After completing this document, confirm:

  [ ] minio Namespace created
  [ ] MinIO Secret created
  [ ] MinIO PVC Bound
  [ ] MinIO Pod Running
  [ ] MinIO Service normal
  [ ] MinIO Console accessible
  [ ] mc can connect to MinIO
  [ ] loki-chunks Bucket created
  [ ] loki-ruler Bucket created
  [ ] loki-admin Bucket created
  [ ] Loki values backed up
  [ ] values-loki-monolithic-minio.yaml written
  [ ] helm template passed
  [ ] helm upgrade successful
  [ ] Loki Pod Running
  [ ] Loki logs have no MinIO connection errors
  [ ] /ready returns ready
  [ ] /metrics accessible
  [ ] Manual push successful
  [ ] query_range can retrieve test logs
  [ ] MinIO Bucket accessible via mc
  [ ] Understand why objects may not appear immediately in MinIO
  [ ] Understand production environments cannot use root user and HTTP plain text configuration

---

## Twenty, Common Misconceptions

### 20.1 Misconception One: Loki Does Not Need PVC After Connecting to MinIO

Not entirely correct.

Even with object storage, Loki may still need local directories for:

  WAL
  Cache
  Temporary data
  Index working directory
  Runtime data

Object storage is primarily used for long-term chunk/index data.

### 20.2 Misconception Two: MinIO Bucket Being Empty Means Loki Failed

Not necessarily.

Loki may not have flushed chunks to object storage yet.

To determine success, check:

    Whether push was successful
    Whether query was successful
    Whether Loki logs show errors
    Whether /ready is normal
    Whether MinIO is accessible
    Whether objects eventually appear

### 20.3 Misconception Three: MinIO Can Be Used in Production with HTTP

Incorrect.

HTTP is only suitable for internal experiments.

Production recommendations include:

    HTTPS
    Strong passwords
    Dedicated users
    Minimum permissions
    Multi-node MinIO
    Monitoring and alerts

### 20.4 Misconception Four: Give MinIO Root User to Loki

Not recommended.

Production should create dedicated Loki users and policies, allowing access only to Loki-related Buckets.

### 20.5 Misconception Five: S3 Configuration Can Only Be Used with AWS

Incorrect.

MinIO is compatible with S3 API, so Loki uses S3-related configurations when connecting to MinIO.

### 20.6 Misconception Six: Object Storage Solves All Loki Issues

Incorrect.

Object storage only addresses long-term storage and scalability issues.

Still need to focus on:

    Label cardinality
    Query range
    Write throttling
    Log volume governance
    Loki self-monitoring
    Compactor
    Ruler
    Gateway
    Alloy collection stability

---

## Twenty-one, Production Environment Deep Understanding

### 21.1 Recommended Production Pipeline

Production recommends:

    Grafana Alloy
      ↓
    Loki Gateway
      ↓
    Loki Writing Component
      ↓
    Object Storage
      ↓
    Grafana Query
      ↓
    Loki Ruler Alerts
      ↓
    AlertManager Notifications

Object storage can be:

    Enterprise MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    Other S3-compatible storage

### 21.2 MinIO Production Monitoring

MinIO itself requires monitoring:

    MinIO Pod Running Status
    MinIO API Availability
    Disk Capacity
    Disk IO
    Bucket Capacity
    S3 Request Volume
    S3 4xx / 5xx Errors
    Request Latency
    Node Health
    Erasure Code Status
    Certificate Expiry

### 21.3 Loki + MinIO Failure Impact

If MinIO is unavailable, it may affect:

    Loki chunk flush
    Historical log queries
    New log writing stability
    Compactor
    Ruler storage
    Query latency

Therefore, object storage cannot be a single point in production.

### 21.4 Retention Period

Loki log retention period needs to combine:

    Loki retention
    Compactor
    Object storage lifecycle
    Compliance requirements
    Cost budget

Note:

    Do not let object storage lifecycle delete data Loki still needs.
    Prioritize understanding Loki's retention configuration and compactor behavior.

---

## Twenty-two, Summary

This article completed the practical implementation of Loki accessing MinIO object storage.

Core pipeline:

    MinIO Deployment
      ↓
    MinIO Service
      ↓
    loki-chunks / loki-ruler / loki-admin Buckets
      ↓
    Loki values configure S3/MinIO
      ↓
    helm upgrade
      ↓
    Loki /ready
      ↓
    Manual push
      ↓
    query_range
      ↓
    MinIO Bucket verification

Key insights:

    Loki can use filesystem storage, but production recommends object storage.
    MinIO accesses Loki through S3-compatible API.
    Loki write success and MinIO immediate object appearance are not fully synchronous concepts.
    Querying recent logs may come from Ingester.
    Object writes may appear only after chunk flush.
    Production environments must consider MinIO high availability, security, permissions, HTTPS, and capacity governance.

After this article, Loki's server has a more production-ready storage foundation.

Next article will enter:

    06-Grafana-Alloy Collecting K8S-Pod Logs in Practice

Focus is:

    Let Kubernetes Pod stdout/stderr logs
      ↓
    Be collected by Alloy
      ↓
    Sent to Loki
      ↓
    Query by namespace, pod, container in Grafana / LogQL

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Configure storage:
  https://grafana.com/docs/loki/latest/setup/install/helm/configure-storage/

- Loki Storage:
  https://grafana.com/docs/loki/latest/configure/storage/

- Loki Configuration Examples:
  https://grafana.com/docs/loki/latest/configure/examples/configuration-examples/

- Loki Helm Chart:
  https://github.com/grafana/loki/tree/main/production/helm/loki

- Install Grafana Loki with Helm:
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Install the monolithic Helm chart:
  https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/

- MinIO Documentation:
  https://min.io/docs/minio/kubernetes/upstream/

- MinIO Client:
  https://min.io/docs/minio/linux/reference/minio-mc.html

- Kubernetes Persistent Volumes:
  https://kubernetes.io/docs/concepts/storage/persistent-volumes/

- Kubernetes Secrets:
  https://kubernetes.io/docs/concepts/configuration/secret/

- Kubernetes Services:
  https://kubernetes.io/docs/concepts/services-networking/service/