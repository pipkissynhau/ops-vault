---
tags: [Kubernetes, Logging, Log Collection, Interview]
---

# Interview Question 5: How to Collect Logs

## Explanation
In Kubernetes, logs are crucial for operations and troubleshooting.  
Container logs generally come from two sources: **stdout/stderr output** and **logs in the node file system**.  
Log collection needs to be centralized for easy storage and analysis.

## Steps / Methods

1. **Node Agent Collection**: Deploy Filebeat or Fluentd on nodes to collect logs from all Pods.  

2. **Container Logs**: The container's stdout/stderr are written by kubelet to `/var/log/pods/`, which can be collected through log agents.  

3. **Centralized Storage**: Send the logs to Elasticsearch, Loki, or Kafka for unified querying and analysis.

### Example Fluentd Configuration

```yaml
<source>
  @type tail
  path /var/log/pods/*.log
  pos_file /var/log/fluentd.pos
  tag kube.*
</source>

<match kube.**>
  @type loki
  url "http://loki:3100/loki/api/v1/push"
</match>
```

## Key Points

- Log collection process: **Collection → Processing → Storage**  
- Container logs can be collected via stdout/stderr or node file system  
- Centralized storage facilitates long-term analysis and alerts  

## Sample Interview Answer

> “In Kubernetes, log collection typically involves two approaches: using agents like Filebeat or Fluentd to collect logs from `/var/log/pods/`, or capturing the container's stdout/stderr outputs, which are stored in the node file system.  
> The collected logs are then sent to centralized storage systems such as Elasticsearch or Loki, allowing operations personnel to perform unified queries, analysis, and set up alerts. This approach ensures log integrity and accessibility.”