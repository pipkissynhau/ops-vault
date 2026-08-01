---
tags: "[Kubernetes, Logs, Log Collection, Interview]"
---

# Interview Question 5: How to Collect Logs

## Overview
In Kubernetes, logs are critical for operations and troubleshooting.  
Container logs typically have two sources: **stdout/stderr output** and **logs in the node filesystem**.  
Log collection requires centralization for convenient storage and analysis.

## Operation Steps / Methods

1. **Node Agent Collection**: Deploy Filebeat or Fluentd on nodes to collect logs from all Pods.  

2. **Container Logs**: Container stdout/stderr is written to `/var/log/pods/` by kubelet, which can be collected via log agents.  

3. **Centralized Storage**: Send logs to Elasticsearch, Loki, or Kafka for unified querying and analysis.

### Fluentd Configuration Example

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

## Key Takeaways

- Log collection process: **Collect → Process → Store**  
- Container logs can be collected via stdout/stderr or node filesystem  
- Centralized storage facilitates long-term analysis and alerting  

## Interview Answer Example

> "In Kubernetes, log collection typically uses two approaches: first, through node agents like Filebeat or Fluentd to collect logs under `/var/log/pods/`; second, collecting container stdout/stderr output, which kubelet writes to the node filesystem.  
> After collection, logs are unified sent to centralized storage like Elasticsearch or Loki, allowing operations teams to query, analyze, and set alerts uniformly. This process ensures log completeness and usability."