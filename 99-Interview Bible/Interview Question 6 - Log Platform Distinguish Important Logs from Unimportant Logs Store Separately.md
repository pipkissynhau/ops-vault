---
tags: "- [Logs, Categorized Storage, K8s, System Logs, Operation Logs, Interview]"
---

# Interview Question 6: How does a log platform distinguish between important and unimportant logs and store them separately

## Explanation
The log platform needs to differentiate based on **log type** (K8s container logs, system logs, operation logs, business logs) and **business priority**, and establish different storage strategies.  
This ensures storage resources are saved while guaranteeing critical business logs remain available for long-term use, facilitating analysis and alerts.

## Method / Operation Steps

1. **Classify Log Types**:
   - **K8s Container Logs**: stdout/stderr or node filesystem `/var/log/pods/`  
   - **System Logs**: Such as /var/log/messages, /var/log/syslog  
   - **Operation Logs**: User operations, audit logs  
   - **Business Logs**: Critical business operations, abnormal events  

2. **Tag / Mark Priority**:
   - Add `severity` or `priority` tags to logs, for example `critical`I don't know.`high`I don't know.`medium`I don't know.`low`  
   - Critical business logs have high priority, ordinary debug logs have low priority  

3. **Storage Strategy**:
   - **High Priority Logs** → Elasticsearch, Loki, or long-term storage  
   - **Medium/Low Priority Logs** → Cold storage, object storage (e.g. MinIO), or compressed archives  
   - Retention time can be set differently based on priority  

4. **Unified Collection and Processing**:
   - Use Fluentd / Filebeat / Logstash to uniformly collect logs from different sources  
   - Route logs to different storage backends based on tags  

## Key Points Summary

- Logs are not limited to K8s, including system logs, operation logs, and business logs  
- Tag logs based on business priority to differentiate importance  
- High priority logs are stored in long-term queryable systems, low priority logs in cold storage or archives  
- Unified collection, unified processing, and categorized storage improve query efficiency and alert accuracy  

## Interview Answer Example

> "In the log platform, I will differentiate importance based on log type and business priority. K8s container logs, system logs, operation logs, and business logs should all be uniformly collected.  
> I will add priority tags to each log, such as critical, high, medium, or low. High priority logs, such as critical business events or abnormal operations, are stored in long-term storage systems like Elasticsearch or Loki to ensure queryability and alerts; medium/low priority logs can be stored in cold storage or compressed archives.  
> This ensures critical logs remain available long-term while saving storage resources, and facilitates operations and auditing."