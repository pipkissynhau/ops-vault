---
tags: [Prometheus, Alerts, Thresholds, Interviews]
---

# Interview Question 2: How to Set Prometheus Alarm Thresholds

## Explanation
Prometheus alerts are based on **Prometheus Rules** (either the PrometheusRule CRD or rules files), where the trigger conditions are defined using `expr`.  
When setting thresholds, it's important to consider the duration to avoid false alarms caused by temporary fluctuations.

## Configuration Example

```yaml
groups:
- name: node_alerts
  rules:
  - alert: NodeHighCPU
    expr: 100 * (1 - avg(irate(node_cpu_seconds_total{mode="idle"}[5m})) > 80
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Node CPU usage exceeds 80%"
      description: "The CPU usage of node {{ $labels.instance }} has exceeded 80% for 5 minutes"
```

## Key Points Summary

1. Use `expr` to define the logic for setting alarm thresholds.
2. The `for` parameter helps prevent false alarms due to momentary fluctuations.
3. `labels` are used to specify the severity of the alert, while `annotations` provide additional descriptive information.

## Sample Interview Answer

> "In Prometheus, alarm thresholds are set by defining rules, with the key field being `expr`, which specifies the trigger condition. For example, an alert could be set for when CPU usage exceeds 80%. To avoid false alarms from temporary spikes, we usually specify a duration, such as 5 minutes, using the `for` parameter. Only if the threshold is maintained for that duration will an alarm be triggered.  
> Additionally, `labels` can be used to define the alert severity (e.g., critical or warning), and `annotations` offer more detailed explanations. Once configured, Prometheus monitors metrics in real-time and triggers alerts as needed."