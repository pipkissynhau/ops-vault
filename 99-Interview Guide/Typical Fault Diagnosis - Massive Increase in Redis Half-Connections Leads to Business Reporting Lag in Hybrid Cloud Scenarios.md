---
tags:
  - #Redis
  - #TCP
  - #Fault Diagnosis
  - #Production Disasters
  - #Hybrid Cloud
  - #Half-Connections
  - #Ops Interviews
  - #Typical Fault Cases
---

# Typical Fault Diagnosis: Massive Increase in Redis Half-Connections Leads to Business Reporting Lag in Hybrid Cloud Scenarios

## I. Fault Background

There is a distributed business system online that calculates the daily or periodic power generation of most wind and water energy facilities across the country.

The overall deployment architecture of the system is as follows:

- The front-end web/business access layer is deployed on the public cloud.
- The Redis cluster and database are deployed on the private cloud.
- The public and private clouds are interconnected through a hybrid cloud network.
- The business system uses the Redis cluster for message processing/message buffering.
- The statistical link requires high real-time performance, and a large amount of device data is reported during peak business hours.

During peak business hours, this system experiences significant lagging, and some data fails to be reported normally, affecting real-time statistics.

---

## II. Fault Symptoms

During peak hours, the following phenomena occurred:

- The entire system lagged.
- Business users reported that data reporting failed or was severely delayed.
- A large number of TCP half-connections accumulated in front of the Redis cluster.
- The peak number of half-connections reached approximately **140,000**.
- The reporting link was blocked. Initially, the business side suspected that **the bandwidth between the public and private clouds was insufficient**.
- After manually clearing the TCP half-connections on-site, the system could recover briefly.

---

## III. Supplementary Business and Technical Background

The core links of such a system can generally be abstracted as follows:

```text  
Wind/water energy facilities  
↓  
Edge gateways/data collection programs  
↓  
Reporting interfaces/access services  
↓  
Redis cluster (as message buffer/message queue)  
↓  
Multiple consumer service instances  
↓  
Statistical services/aggregation services  
↓  
Database/reporting/monitoring boards
```

What such a system fears most is not simply **a service crash**, but rather:

- Inaccurate statistical results.
- Delayed data reporting.
- Missing data for certain periods.
- Difficulty in making up for the missing data.
- Discrepancies between real-time monitoring boards and final reports.

Therefore, if there is congestion at the connection layer during peak hours, the impact on business will be very significant.

---

## IV. Initial Judgment

At first glance, this problem seemed to be a **hybrid cloud network issue**, because:

- The front-end business was deployed on the public cloud.
- Redis and the database were located in the private cloud.
- There was cross-network access between the public and private clouds.
- During peak hours, the system lagged, so it was easy to suspect that the link bandwidth was insufficient.

However, further analysis revealed that:

> This problem was not simply a network bandwidth bottleneck. Instead, it was more like **congestion at the connection establishment layer**, manifested by a large number of TCP half-connections accumulating in front of Redis, preventing businesses from establishing connections and completing data reporting normally.

---

## V. Troubleshooting Ideas and Process

### 1. First, confirm whether it is a network bandwidth bottleneck

Initially, the business side suspected that:

- The bandwidth between the public and private clouds was insufficient.
- During peak hours, the traffic would be fully utilized, causing delays in accessing Redis.

To verify this, **iperf** was used on-site to conduct a traffic testing on the link between the public and private clouds.

#### Troubleshooting Results

- The link throughput was normal.
- No obvious sign of bandwidth saturation was found.
- The possibility of **purely insufficient link bandwidth** was basically ruled out.

#### Conclusion

This step was crucial. It indicated that the problem was not a traditional “network outage” or “insufficient bandwidth”. Instead, further investigation should focus on the following aspects:

- TCP connection establishment capability.
- Redis server access capability.
- Operating system TCP queue parameters.
- Business-side connection establishment model.
- Session pressure at the hybrid cloud boundary devices.

---

### 2. Observe the TCP connection status in front of Redis

Upon further checking the connection status, it was found that a large number of TCP half-connections existed in front of Redis.

This suggested that on-site:

- Clients continuously initiated new connections.
- The Redis server failed to complete connection establishment in a timely manner.
- Half-connections/backlog queues accumulated.
- It became increasingly difficult for new business connections to be established.
- Ultimately, this led to data reporting failures and system lagging.

#### The nature of the fault began to become clear

> During peak hours, the connection establishment capability at the Redis access layer was insufficient, rather than the link transmission capability being inadequate.

---

### 3. Take emergency measures first

Since the business- The efficiency of memory management has decreased.
- Lagging is more likely to occur during peak periods.

These are common optimization measures in a production Redis environment, but they should not be considered the sole cause of this half-open connection issue.

### 4. Correct Understanding

This is not a "compilation or installation issue with Redis," but rather:

- The system parameters were not optimized during Redis' production deployment.
- Official warnings were not addressed before going live.
- The OS and Redis parameters were not aligned.
- There was a lack of production environment standards.

---

## VIII. Root Cause Analysis

### 1. Direct Technical Causes

The host where Redis is running did not optimize its system kernel parameters for production use, resulting in insufficient backlog capacity.

The key evidence is:

- The Redis configuration intended a backlog of 511.
- However, Linux's `somaxconn` setting was only 128.
- As a result, the actual listening queue capacity was too small.
- During peak connection periods, it could not handle a large number of sudden connections.
- This led to a significant accumulation of TCP half-open connections.

---

### 2. Secondary Risk Factors

#### Official warnings in the Redis production environment were ignored

##### `overcommit_memory=0`

This could cause:

- Backlog persistence failures in the background.
- Fork failures.
- Increased memory fluctuations under high load conditions.

##### THP was not disabled

This could result in:

- Poorer memory allocation efficiency.
- More significant latency and performance fluctuations in Redis.
- Reduced stability under high-load scenarios.

While these factors were not the direct cause of the half-open connection issue, they did contribute to increased instability during peak periods.

---

### 3. Possible Business-Side Issues

Although the problems with Redis/OS parameters are clear, attributing the entire issue solely to Redis would be incomplete.

From an architectural perspective, the business side could also be responsible for the following issues:

#### (1) Insufficient Redis connection reuse

During peak periods, if the business side did not use a connection pool correctly or frequently established and disconnected connections, it could lead to:

- A surge in short-lived connections.
- A sudden increase in the number of new connections.
- Overloading of the Redis access layer.

#### (2) Improper configuration of the connection pool

For example:

- The connection pool size was too small.
- Too many new connections were created during peak periods.
- Inappropriate timeout and retry strategies were implemented.
- Some instances experienced excessive retry attempts.

#### (3) Frequent cross-cloud calls to Redis

When public cloud services frequently accessed private cloud Redis, it could exacerbate the following issues:

- Connection establishment delays.
- Additional overhead for cross-cloud handshaking.
- Increased session pressure on border devices.
- Vulnerability of the connection establishment layer under high concurrency.

#### (4) Unhealthy business traffic patterns during peak periods

For example:

- Sudden surge in reports.
- Concentrated writing activities during certain time periods.
- Lack of load mitigation and throttling measures.
- Mixing of MQs and caches, leading to excessive pressure on Redis.

---

## IX. Final Root Cause Identification

### It is recommended to describe it this way in the review:

> The direct technical cause of this failure was that the host where Redis was running did not optimize its system kernel parameters according to production standards. With `net.core.somaxconn` set at only 128, the configured `tcp-backlog` value could not take effect. As a result, during peak connection periods, the capacity of the connection establishment queue was insufficient, leading to a significant accumulation of TCP half-open connections. This caused congestion in the Redis access layer and lagging in business report processing.
> Additionally, the host had production-risk factors such as `overcommit_memory=0` and THP not being disabled, which further increased the performance fluctuations and stability risks under high load conditions.
> From a system-wide perspective, the business side's Redis access model might also have issues such as insufficient connection reuse, short-lived connection storms, or improper connection pool configuration. These factors combined to exacerbate this failure in a mixed-cloud environment.

---

## X. Summary of Emergency Measures

The emergency measures taken at the scene of this failure included:

- Using iperf to identify any bottlenecks in network link bandwidth.
- Identifying and manually clearing the large number of TCP half-open connections in front of Redis.
- Adjusting the operating system kernel's network parameters.
- Improving the short-term capacity of the Redis access layer.

### Nature of these Measures

These actions were:

- **Emergency measures to stabilize the situation**
- **Quick steps to restore business operations**
- **Measures to alleviate access congestion**

They were not intended as long-term architectural solutions.

---

## XI. Follow-up Optimization Suggestions

### 1. Operating System Layer

Key optimizations include:

- Adjusting `net.core.somaxconn`.
- Changing `net.ipv4.tcp_max_syn_back---

## Chapter Fourteen: Additional Common Commands for Investigating TCP States

### 1. View All TCP Listening States

`ss -ntl`

### 2. View Connection States Related to Redis Port 6379

`ss -ant | grep 6379`

### 3. Count the Number of Each Type of TCP State

`ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c`

### 4. Focus on Semi-Connections (SYN-RECV)

`ss -ant | grep SYN-RECV | wc -l`

### 5. Check for a Large Number of TIME-WAIT States

`ss -ant | grep TIME-WAIT | wc -l`

### 6. View Kernel Backlog Related Parameters

`sysctl net.core.somaxconn`  
`sysctl net.ipv4.tcp_max_syn_backlog`

### 7. Look for Backlog Alerts in Redis Startup Logs

`grep -i backlog /apps/redis/logs/redis.log`

---

## Chapter Fifteen: Interview Answer Templates

### Version One: Concise Version

I encountered a Redis failure in a hybrid cloud environment. The front-end services were deployed on the public cloud, while Redis and the database were located in the private cloud. During peak hours, the system experienced lagging and data reporting issues. Initially, the team suspected that it was due to insufficient cross-cloud bandwidth. I first used iperf to test the network throughput and confirmed that there was no bandwidth issue. Subsequently, I noticed a large number of TCP semi-connections in front of Redis, with a peak of around 140,000. This indicated that the problem lay in the connection establishment phase. By reviewing the Redis startup logs, I found that although Redis was configured to use a backlog size of 511, this value was actually limited by the system's `somaxconn` setting of 128, resulting in insufficient capacity for the connection establishment queue during peak hours. The direct cause was the lack of optimized production-grade kernel parameters on the host machine. I resolved the issue temporarily by clearing semi-connections and optimizing kernel settings, and later implemented long-term solutions involving business connection pooling, Redis configuration adjustments, and improving the hybrid cloud access model.

### Version Two: Detailed Version

I dealt with a typical Redis connection layer failure in a scenario where data from wind and water power generation facilities across the country was being collected. The front-end web services were on the public cloud, while the Redis cluster and database were in the private cloud. During peak hours, there was a significant increase in data reporting volume, causing system lagging and data failures. At first, we suspected that the issue stemmed from insufficient bandwidth between the public and private clouds. However, after testing the network throughput with iperf, we ruled out this possibility and focused on connection establishment capabilities. Further investigation revealed approximately 140,000 TCP semi-connections in front of Redis, indicating congestion at the connection layer. The Redis startup logs contained a critical warning: although Redis was set to use a backlog size of 511, the system's `somaxconn` parameter limited this value to 128, significantly reducing the queue capacity. This directly corresponded to the observed semi-connection accumulation. The root cause was the lack of optimized production-grade kernel parameters on the host machine, which resulted in insufficient connection establishment queue capacity during peak hours. Additional factors contributing to the issue were the setting `overcommit_memory=0` and the disabled THP mechanism, which also impacted Redis stability. I also considered potential issues with the business-side connection pooling or short-connection storms, which exacerbated the problem in the hybrid cloud environment. My initial response was to clear semi-connections and optimize kernel settings to restore service availability, followed by long-term improvements involving Redis configuration, OS parameters, business connection reuse, and architectural adjustments.