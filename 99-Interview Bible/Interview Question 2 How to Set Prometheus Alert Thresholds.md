---
tags: "[Prometheus, Alerts, Threshold, Interview]"
---

# Interview Question 2: How to Set Prometheus Alert Thresholds

## Explanation
Prometheus alerts are based on **Prometheus Rule** (PrometheusRule CRD or rules file), defined by `expr` to specify trigger conditions.  
Threshold settings should consider duration to avoid false triggers caused by transient fluctuations.

## Configuration Example

```yaml
groups:
- name: node_alerts
  rules:
  - alert: NodeHighCPU
    expr: 100 * (1 - avg(irate(node_cpu_seconds_total{mode="idle"}[5m]))) > 80
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Nodes CPU Overuse 80%"
      description: "Nodes {{ $labels.instance }} CPU Overuse 80% Ongoing 5 min"
```

## Key Points Summary

1. Use `expr` to define alert threshold logic  
2. `for` parameter prevents false triggers from transient fluctuations  
3. `labels` marks alert severity level, `annotations` provides alert description information

## Interview Answer Example

> "Prometheus alerts are set by defining rules, with the key field being `expr`, which defines the trigger condition, such as CPU usage exceeding 80%. To avoid false triggers from transient fluctuations, we typically set `for` duration, like 5 minutes, so alerts only trigger when the threshold is exceeded continuously.  
> Meanwhile, you can use `labels` to define alert severity level, such as critical or warning, and use `annotations` to provide description information. After configuration, Prometheus will calculate metrics in real-time based on the rules, and trigger alerts and send them to Alertmanager once conditions are met."