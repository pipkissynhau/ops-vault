---
tags: "[Prometheus, Alertmanager, Alert Notifications, Interview]"
---

# Interview Question 3: How Does Prometheus Alert Notification Work (Notification)

## Explanation
Prometheus sends alerts via **Alertmanager**, which can notify through email, DingTalk, Slack, or Webhook channels.  
Alert notifications can be grouped by `severity` or `alertname`, enabling flexible routing to different receivers.

## Configuration Example

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  receiver: 'email-alert'

receivers:
- name: 'email-alert'
  email_configs:
  - to: 'ops@example.com'
    from: 'prometheus@example.com'
    smarthost: 'smtp.example.com:587'
    auth_username: 'prometheus'
    auth_password: 'password'
```

## Key Points Summary

1. Prometheus alerts are sent via **Alertmanager**  
2. Notifications can be routed to different channels based on `severity` or `alertname`  
3. Supports multiple notification methods: email, DingTalk, Slack, Webhook, etc.  

## Interview Answer Example

> "Prometheus alert notifications are not sent directly, but through Alertmanager. We can configure receivers in Alertmanager, such as email, DingTalk, or Slack, and route alerts based on severity level or alert name. When Prometheus detects metrics reaching thresholds, it triggers rules, and Alertmanager sends the alerts to the configured receivers. This ensures timely delivery and clear categorization of alerts, facilitating efficient operations handling."