---
tags: [logging, classified storage, Kubernetes, system logs, operation logs, interview]
---

# Interview Question 6: How Does a Logging Platform Distinguish Between Important and Unimportant Logs and Store Them Accordingly

## Explanation
A logging platform needs to distinguish between important and unimportant logs based on **log type** (Kubernetes container logs, system logs, operation logs, business logs) and **business priority**, and implement different storage strategies accordingly.  
This approach helps to optimize storage resources while ensuring that critical business logs are available for a long time, facilitating analysis and alerts.

## Methods / Steps

1. **Classify Log Types**:
   - **Kubernetes Container Logs**: stdout/stderr or node file system `/var/log/pods/`
   - **System Logs**: For example, /var/log/messages, /var/log/syslog
   - **Operation Logs**: User operations, audit logs
   - **Business Logs**: Critical business activities, exception events

2. **Add Tags / Priority Labels**:
   - Assign `severity` or `priority` tags to logs, such as `critical`, `high`, `medium`, `low`
   - Critical business logs should have higher priorities than regular debugging logs

3. **Storage Strategies**:
   - **High-Priority Logs**: Store in Elasticsearch, Loki, or long-term storage solutions
   - **Medium- and Low-Priority Logs**: Use cold storage, object storage (e.g., MinIO), or compressed archiving
   - Retention periods can vary depending on priority levels

4. **Unified Collection and Processing**:
   - Use tools like Fluentd, Filebeat, or Logstash to collect logs from various sources
   - Route logs to different storage targets based on their tags

## Key Points Summary

- Logs include not only Kubernetes logs but also system logs, operation logs, and business logs.
- Assign tags based on business priority to distinguish between important and unimportant logs.
- Store high-priority logs in long-term accessible systems; store low-priority logs in cold storage or archives.
- Implement unified collection, processing, and classified storage to improve query efficiency and alert accuracy.

## Sample Interview Answer

> “In a logging platform, I would classify logs based on their type and business priority. All types of logs—Kubernetes container logs, system logs, operation logs, and business logs—should be collected uniformly.
> I would assign priority tags to each log, such as critical, high, medium, or low. High-priority logs, like critical business events or errors, would be stored in Elasticsearch or Loki for long-term retention and easy access for alerts and analysis. Medium- and low-priority logs could be archived or stored in cost-effective solutions.
> This approach ensures that important logs are always available while optimizing storage usage and making it easier for operations teams and auditors to manage log data.”