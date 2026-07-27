---
tags: [Prometheus, Alertmanager, Notification, Interview]
---

# Interview Question 3: How Does Prometheus Notify About Alerts (Notification)

## Explanation
Prometheus sends alerts through **Alertmanager**, which can notify via channels such as email, DingTalk, Slack, or Webhook.  
Alert notifications can be grouped by `severity` or `alertname` and flexibly routed to different recipients.

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

1. Prometheus alerts are sent through **Alertmanager**.
2. Notifications can be routed based on `severity` or `alertname`.
3. Multiple notification methods are supported, including email, DingTalk, Slack, and Webhook.

## Sample Interview Answer

> “Prometheus does not send alerts directly; instead, it uses Alertmanager to handle the notifications. We can configure receivers in Alertmanager, such as email, DingTalk, or Slack, and we can also group notifications by their severity or name. This way, when Prometheus detects that a metric has reached a threshold, it triggers the relevant rule, and Alertmanager then sends the notification to the designated channel. This approach ensures that alerts are delivered promptly and are organized clearly, making them easier for operations personnel to process.”