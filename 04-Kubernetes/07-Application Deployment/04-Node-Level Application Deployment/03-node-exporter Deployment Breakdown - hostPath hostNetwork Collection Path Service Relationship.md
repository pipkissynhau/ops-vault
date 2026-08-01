# 03-node-exporter Deployment Breakdown: hostPath, hostNetwork, Collection Paths and Service Relationship

## Document Notes
- Document Positioning: Breakdown of node-exporter deployment in Kubernetes at the node level
- Applicable Stage: After understanding DaemonSet basics and the differences between Deployment, StatefulSet, and DaemonSet workload models, entering specific node-exporter deployment understanding
- Recommended Path: `04-Kubernetes/07-Apply deployment/04-Application deployment at node level/03-node-exporter Deployment of dismantling:hostPathI don't know.hostNetworkand the collection path Service Relations`

## Tags
#Kubernetes #DaemonSet #node-exporter #Prometheus #hostPath #hostNetwork #Monitor #ApplicationAtNodeLevel #Service #ApplyDeployment #Clouds. #Transport

---

## I. Why Decompose node-exporter Separately

node-exporter is one of the most typical and suitable entry-level cases for node-level applications.

The previous article established the fundamental differences between the three workload models:

- Deployment focuses more on replica counts
- StatefulSet focuses more on member relationships
- DaemonSet focuses more on node coverage

node-exporter perfectly aligns the DaemonSet mainline to a specific component.

This article's focus isn't on the full Prometheus ecosystem, but rather answering these practical questions:

- Why is node-exporter typically deployed as a DaemonSet?
- Why does it often need to mount host directories?
- Why does it frequently use `hostNetwork`?
- What metrics does it actually collect?
- What role does Service play here?
- What are the key differences in deployment between such components and regular business applications?

---

## II. What Exactly is node-exporter

node-exporter is a common node metric collection component in the Prometheus ecosystem.

Its main responsibility is:

> **Expose native system metrics of the node as Prometheus-parsable metrics interface.**

### Common Collection Contents Include

- CPU usage
- Memory usage
- File system capacity
- Disk I/O
- Network interface statistics
- System load
- Boot time
- Some kernel and system runtime information

### What It Does Not Primarily Do

node-exporter's focus is not:

- Collecting business logs
- Collecting application traces
- Collecting custom business metrics
- Directly analyzing container application errors

It leans more toward:

> **Collecting resource metrics at the host and node levels.**

---

## III. Why node-exporter is Suitable for DaemonSet

The core of this question isn't about "officially written this way," but rather the component's inherent responsibilities.

### 1. It Collects Node Native Metrics
Since it collects node metrics, each node needs its own collector.

### 2. Its Goal is Node Coverage, Not Replica Count
If the cluster has 6 nodes, the expectation is typically:

- 1 node-exporter per node

Rather than:

- Running 2 or 3 replicas total

### 3. New Nodes Automatically Get Collectors
When a new node is added, DaemonSet automatically deploys a Pod on it, which aligns perfectly with node-exporter's goals.

### 4. Nodes Leaving the Cluster No Longer Need This Instance
If a node leaves the cluster, the corresponding node-exporter instance loses its purpose.

### Operations Understanding Focus
node-exporter is not a "monitoring service replica," but rather:

> **A metrics proxy on each node.**

---

## IV. Let's Look at a Teaching-Style node-exporter Example

Below is a simplified but sufficient example to understand the structure.

### DaemonSet Example

    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: node-exporter
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: node-exporter
      template:
        metadata:
          labels:
            app: node-exporter
        spec:
          hostNetwork: true
          hostPID: true
          tolerations:
            - operator: Exists
          containers:
            - name: node-exporter
              image: prom/node-exporter:v1.8.1
              ports:
                - containerPort: 9100
                  hostPort: 9100
              args:
                - --path.procfs=/host/proc
                - --path.sysfs=/host/sys
                - --path.rootfs=/host/root
              volumeMounts:
                - name: proc
                  mountPath: /host/proc
                  readOnly: true
                - name: sys
                  mountPath: /host/sys
                  readOnly: true
                - name: root
                  mountPath: /host/root
                  readOnly: true
          volumes:
            - name: proc
              hostPath:
                path: /proc
            - name: sys
              hostPath:
                path: /sys
            - name: root
              hostPath:
                path: /

### Service Example /think

apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    app: node-exporter
  ports:
    - name: metrics
      port: 9100
      targetPort: 9100

This example is not a complete production manifest, but it's very suitable for breaking down several key points of node-exporter.

---

## V. What is the most critical part of this DaemonSet

If we focus only on the most important items, we can first look at:

- `kind: DaemonSet`
- `hostPath`
- `hostNetwork`
- `args`
- `Service`

These fields basically explain the deployment model of node-exporter.

---

## VI. Why does node-exporter use hostPath

### 1. What is hostPath
`hostPath` indicates directly mounting a directory from the host into the Pod.

In the node-exporter example, common mounts include:

- `/proc`
- `/sys`
- `/`

### 2. Why it must access host directories
Because node-exporter wants to collect node-level status, not container sandbox internal state.

Without reading host directories, node-exporter might only see the container's internal perspective, not the actual node resource status.

### 3. Typical Mount Examples

    volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /

Corresponding mounts in the container:

    volumeMounts:
      - name: proc
        mountPath: /host/proc
        readOnly: true
      - name: sys
        mountPath: /host/sys
        readOnly: true
      - name: root
        mountPath: /host/root
        readOnly: true

### Operations Understanding Focus
This is not "mounting several directories arbitrarily for the container", but expressing:

> **Let the container read node metrics from the host perspective.**

---

## VII. Why mount `/proc`

### What does `/proc` provide
The host's `/proc` is a very core runtime information directory in Linux, containing a lot of:

- CPU-related information
- Process information
- Memory information
- Kernel state information
- Network stack-related information

### Why node-exporter needs it
Many node metrics are not generated out of nowhere, but are read and calculated from `/proc`.

### Example Parameters

    --path.procfs=/host/proc

This indicates telling node-exporter:

- Do not read the container's own `/proc`
- Read the mounted host's `/proc`

### Operations Understanding Focus
Without this layer, the metrics collected by node-exporter might be incomplete, or even biased toward the container itself, rather than the entire node.

---

## VIII. Why mount `/sys`

### What does `/sys` provide
The host's `/sys` usually contains:

- Device information
- File system information
- Kernel-exported system state
- Some hardware and network interface-related information

### Why node-exporter needs it
Many node-level resource metrics depend on content from `/sys` for supplementation or verification.

### Example Parameters

    --path.sysfs=/host/sys

This indicates node-exporter reads the host's `/sys`.

### Operations Understanding Focus
`/proc` and `/sys` can be understood as the two most common basic information channels for node-exporter to read node status.

---

## IX. Why mount the host root directory `/`

### Why mount the root directory
Here it's usually not for "grabbing all files arbitrarily", but to let node-exporter identify from the host perspective:

- File system mount points
- Disk usage
- Root file system and other mounted volumes

### Example Parameters

    --path.rootfs=/host/root

Indicates mapping the host root directory to the container's `/host/root`, and informing node-exporter to use it as the host root path reference.

### Operations Understanding Focus
This step is mainly to let node-exporter see "node file systems", not just the container file systems.

---

## X. Why these mounts are usually read-only

In the example, the mount syntax is always:

    readOnly: true

### Reason
node-exporter's responsibility is collection, not modifying host state.

Therefore, a more reasonable approach is:

- Read-only mount host directories
- Read metrics only
- No write capability to host directories

### Operations Understanding Focus
Node-level collection components usually follow a basic principle:

> **Read-only when possible, never give write permissions.**

This aligns with the principle of least privilege and reduces the risk of accidental operations.

---

## XI. Why node-exporter often uses hostNetwork

### 1. What is hostNetwork
`hostNetwork: true` indicates the Pod uses the host's network namespace directly, not the default Pod-independent network.

### 2. Common reasons in node-exporter
This is usually done for several considerations:

- Expose port 9100 more directly on the node network
- Facilitate Prometheus to scrape by node address
- Reduce some network layer forwarding complexity
- Closer to the node agent-style component operation mode

### Example

    hostNetwork: true

### 3. This doesn't mean "must always be written this way"
Some environments may also not use `hostNetwork`, but adopt other scraping and exposure methods.

### Operations Understanding Focus
Currently, the more important thing is to understand:

> **Node-level agents often need to be closer to the host network.**

---

## XII. Why hostPort often appears in examples

The example writes: /think

ports:
  - containerPort: 9100
    hostPort: 9100

### What does this mean
Means:
- Container listens on 9100
- Node host also uses 9100

### Why is it written this way
Because each node typically runs only one node-exporter, so:
- Node host 9100 port is used by the node-exporter on this node
- Prometheus can directly scrape metrics via node IP + 9100

### Notes
Once using `hostPort`, you need to be aware of:
- Port conflict
- Whether other components on the node are already using 9100

### Operational Understanding Focus
When `hostNetwork` and `hostPort` appear, it usually means the component is running closer to the host network.

---

## Thirteen, Why node-exporter Sometimes Uses hostPID

Example shows:

    hostPID: true

### Its Meaning
Indicates the Pod uses the host's PID namespace.

### Current Stage Understanding
For node-exporter, the core significance of this configuration isn't in deep Linux namespace details, but in understanding:

> **Node-level agents sometimes need to run closer to the host environment.**

### Notes
Like `hostNetwork`, this configuration also means higher host visibility, so it should be used cautiously and with understanding of its security boundaries.

---

## Fourteen, What Role Do args Play in node-exporter

Example startup parameters are:

    args:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/host/root

### Essence of These Parameters
Not just "randomly writing paths", but telling node-exporter:

- Where the host's proc is
- Where the host's sys is
- Where the host's root path is mapped in the container

### What Does This Mean
This means node-exporter doesn't automatically know host paths, but:

- The container brings in host directories via `hostPath`
- Then tells the program where to read these directories

### Operational Understanding Focus
This is a common pattern for many node-level components:

> **Not only mount host directories, but explicitly tell the program how to use these directories.**

---

## Fifteen, What Role Does Service Play in node-exporter

This is often confused with MySQL's Service.

### What is MySQL's Service More Biased Toward
MySQL's Service is more biased toward:
- Business access entry
- Fixed database connection address

### What is node-exporter's Service More Biased Toward
node-exporter's Service is more biased toward:
- Metrics scraping entry
- Monitoring system discovery target
- Unified exposure of metrics port

### Example

    apiVersion: v1
    kind: Service
    metadata:
      name: node-exporter
      namespace: monitoring
    spec:
      selector:
        app: node-exporter
      ports:
        - name: metrics
          port: 9100
          targetPort: 9100

### What Does This Mean
It's not for business access to node-exporter, but to make Prometheus more convenient for discovery and scraping node-exporter metrics.

### Operational Understanding Focus
Although both are Service, the role changes with component type:

- MySQL: Biased toward business entry
- node-exporter: Biased toward monitoring scraping entry

---

## Sixteen, Why node-exporter Often Tolerates More Nodes

Example shows:

    tolerations:
      - operator: Exists

### What Does This Mean
Indicates this DaemonSet is more tolerant of taints, common purposes include:

- Let it schedule to control plane nodes
- Avoid certain nodes from being unable to run the collector due to taints
- Ensure more complete node metric coverage

### Why This is Common
Because node-exporter's goal is usually:

> **To cover as many nodes as possible**

Including:
- Worker nodes
- Control plane nodes
- Special role nodes

### Operational Understanding Focus
Node-level collection components usually emphasize "full coverage", so scheduling constraints often tend to be wider than ordinary business applications.

---

## Seventeen, A More Practical Inspection Method

After deployment, common inspection methods are as follows.

### 1. Check DaemonSet Status

    kubectl get ds -n monitoring
    kubectl describe ds node-exporter -n monitoring

Focus on:
- Expected node count
- Scheduled Pod count
- Available Pod count

### 2. Check Pod Distribution

    kubectl get pod -n monitoring -o wide

Focus on:
- Whether each node has a node-exporter
- Whether Pods are distributed on expected nodes

### 3. Check Service

    kubectl get svc -n monitoring
    kubectl describe svc node-exporter -n monitoring

### 4. Check Pod Logs

    kubectl logs -n monitoring <node-exporter-pod-name>

### 5. Verify metrics Interface
If network allows, you can access:

    http://<node-ip>:9100/metrics

Or verify via port forwarding:

    kubectl port-forward -n monitoring <node-exporter-pod-name> 9100:9100

Then access locally:

    http://127.0.0.1:9100/metrics

---

## Eighteen, What is Most Often Overlooked for node-exporter This Type of Node-Level Application

### 1. Mistaking It for a Regular Service Replica
Only care about Pod startup, ignoring whether node coverage is achieved.

### 2. Not Understanding the Meaning of hostPath
Seeing many host directories mounted, just mechanically memorize, instead of understanding it's for collecting node host information.

### 3. Not Understanding the Role Change of Service
Treat it as a regular business entry, instead of a monitoring scraping entry.

### 4. Not Understanding Why Closer to Host is Needed
For `hostNetwork`, `hostPID`, `hostPort`, only memorize "these fields need to be written", but don't understand they express "host visibility".

### 5. Ignoring Scheduling Coverage Issues
Not realizing that the key for node-level applications isn't replica count, but node coverage rate.

---

## Nineteen, The Most Important Several Understandings

### 1. The goal of node-exporter is not to provide business services, but to provide node metrics
This is the first understanding.

### 2. DaemonSet is suitable for node-exporter, because it pursues one instance per node
This is the second understanding.

### 3. The essence of hostPath is to allow containers to read data from the host machine's perspective
This is the third understanding.

### 4. hostNetwork / hostPort / hostPID express a more host-machine-aligned runtime mode
This is the fourth understanding.

### 5. The Service of node-exporter is more oriented toward monitoring scrape entry points, rather than business entry points
This is the fifth understanding.

---

## Twenty, Stage Summary

node-exporter is a natural case for understanding node-level application deployment.

Through it, the main line of DaemonSet can be addressed to several very specific questions:

- Why is one instance needed per node
- Why is mounting host directories necessary
- Why is closer alignment with host machine networking needed
- Why is the Service here a monitoring scrape entry point
- Why such components should not be understood as ordinary business replicas

This article truly aims to establish an overall sense:

> **The focus of node-level applications is not business service replicas, but host machine capability collection and node coverage.**

---

## Twenty-one, Keyword Quick Notes

- node-exporter: Node metrics collection component
- DaemonSet: Node coverage controller
- hostPath: Mount host directory
- `/proc`: Host machine runtime information
- §§inline_37§s§: Host machine system and device information
- `hostNetwork`: Use host machine networking
- `hostPort`: Occupy host machine ports
- Service: Monitoring scrape entry point
- Node coverage: Each node has an instance

---

## Twenty-two, Operations Extension Understanding

From the workload model perspective, node-exporter very well demonstrates the difference between node-level applications and business applications.

Business applications are more concerned with:
- Traffic
- Replica count
- Deployment strategy

Stateful applications are more concerned with:
- Data
- Identity
- Recovery and membership relations

While node-exporter type node-level applications are more concerned with:
- Host machine visibility
- Metric collection
- Node coverage
- Whether they run closely aligned with nodes

Therefore, understanding the deployment mode of node-exporter essentially lays the foundation for further understanding:

- Fluent Bit / Filebeat
- Security Agent
- CNI / CSI node-side components

---

## References
- Kubernetes DaemonSet official documentation
- node-exporter official documentation
- Prometheus official documentation

---

## Next Day Suggestions
Next article suggestion to organize:

[[04-Why DaemonSet is Commonly Used for Log Collectors Fluent Bit Filebeat Node-Level Deployment Strategy]]