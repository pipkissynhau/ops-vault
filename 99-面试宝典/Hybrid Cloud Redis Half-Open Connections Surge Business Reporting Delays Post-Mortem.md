---
tags:
  - #Redis
  - #TCP
  - #FaultRewind
  - #ProductionAccidents
  - #MixedClouds
  - #Semi-connection
  - #TransportInterview
  - #TypicalFailureCases
---

# Root Cause Analysis of Severe Redis Half-Connection Surge Causing Business Reporting Latency in Hybrid Cloud Scenarios

## I. Background

There is a distributed business system online, used for statistics on the daily power generation or power generation over specific time periods of most wind and water equipment nationwide.

The overall system architecture is as follows:

- Frontend Web / Business Access Layer deployed in public cloud
- Redis Cluster and database deployed in private cloud
- Public cloud and private cloud interconnected via hybrid cloud network
- Business system uses Redis Cluster for message processing/message buffering
- Statistics pipeline has high real-time requirements, with large volumes of device data reporting during peak business hours

The system experienced noticeable latency during peak business hours, with some data failing to report normally, affecting real-time statistics.

---

## II. Fault Symptoms

During peak hours, the following phenomena occurred:

- System-wide latency
- Business side reported data reporting failures or severe delays
- Large number of TCP half-connections appeared before Redis Cluster
- Half-connection peak count reached approximately **140,000**
- Reporting pipeline was blocked, business side initially suspected **insufficient bandwidth between public cloud and private cloud**
- On-site manual cleanup of TCP half-connections temporarily restored system functionality

---
## III. Business and Technical Background Supplement

This type of system's core pipeline can generally be abstracted as:

```text  
Wind engine/Water equipment  
↓  
Edge Gateway / Data collection procedures  
↓  
Report interface / Access services  
↓  
Redis Cluster (as a buffer for messages) / Message Queue)  
↓  
Multiple examples of consumer services  
↓  
Statistical services / Syndication services  
↓  
Database / Report / Watch.
```
Such systems are most afraid not of "service failure", but rather:

- Inaccurate statistical results
    
- Data reporting delay
    
- Data missing during certain periods
    
- Difficult data reconciliation
    
- Inconsistency between real-time dashboards and final reports
    
  

Therefore, if Redis experiences connection layer congestion during peak hours, the business impact will be very significant.

---

## IV. First Judgment

At first glance, this issue appears to be a "hybrid cloud network problem", because:

- Frontend business deployed in public cloud
    
- Redis and database in private cloud
    
- Cross-network access between public and private clouds
    
- System latency during peak hours, initial suspicion was likely **insufficient link bandwidth**
    

However, further analysis revealed:

> This issue was not a simple network bandwidth bottleneck, but more likely **connection establishment layer congestion**, manifested as a large accumulation of TCP half-connections before Redis, causing business to fail to establish normal connections and complete reporting.

---

## V. Troubleshooting Process and Analysis

### 1. First confirm if it's a network bandwidth bottleneck

The business side initially suspected:

- Insufficient bandwidth between public cloud and private cloud
    
- High traffic during peak hours filled the link, causing Redis access latency
    
  

To verify this, on-site **iperf** was used to perform traffic testing between public cloud and private cloud.

#### Investigation Results

- Link throughput was normal
    
- No obvious bandwidth saturation observed
    
- Basically ruled out the direction of "pure link bandwidth insufficiency"
    

#### Conclusion

This step was critical, indicating the issue was not a traditional "network unavailability" or "bandwidth insufficiency", and should continue investigation in the following directions:

- TCP connection establishment capability
    
- Redis server access capacity
    
- Operating system TCP queue parameters
    
- Business-side connection establishment model
    
- Session pressure on hybrid cloud boundary devices
    
  

---

### 2. Observe TCP connection status before Redis

After further examination of connection status, a large number of TCP half-connections were found before Redis.

This indicates the scene was more likely:

- Clients continuously initiating new connections
    
- Redis server failing to complete connection acceptance in time
    
- Accumulation of half-connections/backlog queue
    
- New business connections increasingly difficult to establish
    
- Eventually leading to reporting failures and system latency
    
  

#### The root cause began to narrow down to

> Insufficient connection establishment capability at Redis access layer during peak hours, rather than insufficient link transmission capability.

---

### 3. First perform emergency mitigation

With the business already affected, on-site emergency measures were taken:

- Manually clean up TCP half-connections
    
- Optimize operating system kernel TCP-related parameters
    
- Alleviate connection backlog before Redis
    
  

#### Emergency Results

- Half-connection count decreased
    
- Redis access congestion was somewhat alleviated
    
- Business reporting capability recovered
    
- System latency significantly reduced
    
  

#### Note

This step was an **emergency mitigation measure**, aiming to restore business functionality first, not equivalent to completing root cause resolution.

---

## VI. Key Evidence

Subsequent review of Redis startup logs revealed important warnings:

[root@centos8 ~]# redis-server /apps/redis/etc/redis.conf  
27569:C 16 Feb 2020 21:18:20.412 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo  
27569:C 16 Feb 2020 21:18:20.412 # Redis version=5.0.7, bits=64,  
commit=00000000, modified=0, pid=27569, just started  
27569:C 16 Feb 2020 21:18:20.412 # Configuration loaded  
27569:M 16 Feb 2020 21:18:20.413 * Increased maximum number of open files to 10032 (it was originally set to 1024).  
27569:M 16 Feb 2020 21:18:20.414 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.  
27569:M 16 Feb 2020 21:18:20.414 # Server initialized  
27569:M 16 Feb 2020 21:18:20.414 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition.  
27569:M 16 Feb 2020 21:18:20.414 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel.  
27569:M 16 Feb 2020 21:18:20.414 * Ready to accept connections

Meanwhile, through `ss -ntl`, it was visible that Redis actual listening queue was only 128: /think

[root@centos8 ~]# ss -ntl  
State   Recv-Q Send-Q Local Address:Port  Peer Address:Port  
LISTEN  0      100    127.0.0.1:25        *:*  
LISTEN  0      128    127.0.0.1:6379      *:*  
LISTEN  0      128    *:22                *:*  
LISTEN  0      100    [::1]:25            [::]:*  
LISTEN  0      128    [::]:22             [::]:*

This indicates:

### 1. Redis wants to use `tcp-backlog=511`

But the system kernel parameter `net.core.somaxconn=128` causes the Redis backlog configuration to fail to take effect.

### 2. During peak periods with massive connection arrivals

The Redis server's connection establishment queue is too small to handle high concurrency connection requests, leading to:

- Half-open connection backlog
    
- Backlog overflow
    
- Accept not timely
    
- Difficulty establishing new connections
    
    

### 3. Other two warnings

- `overcommit_memory=0`
    
- `THP enabled`
    

Although not the direct cause of the half-open connection surge, they increase Redis's high-load jitter and performance risks, indicating a non-compliant production deployment of Redis.

---

## Seven. Correct Understanding of These Three Warnings

### 1. `somaxconn=128` is too small —— Highly related to this fault

This is the most critical direct evidence:

WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

Meaning:

- Redis wants to set `tcp-backlog` to `511`
    
- But the Linux kernel parameter `net.core.somaxconn` is only `128`
    
- Redis's listening queue upper limit is ultimately restricted by the OS
    
- During peak connection establishment, the queue easily gets filled
    
    

This aligns with the on-site phenomenon:

1. Business concurrency increases during peak periods
    
2. Redis front connection requests surge
    
3. Backlog queue capacity is too small
    
4. Half-open connections quickly accumulate
    
5. Redis access layer congestion
    
6. Business reports fail, system stalls
    
    

### 2. `overcommit_memory=0` —— Production risk item, not the main cause of this half-open connection surge

This mainly affects:

- `BGSAVE`
    
- `BGREWRITEAOF`
    
- Forking child processes
    
- Persistence failure or jitter when memory is tight
    
    

It leans more toward:

- Stability hazards
    
- Risk amplification under high load
    
    

### 3. THP is enabled —— Performance degradation factor, not the most direct cause this time

THP causes:

- Redis latency jitter
    
- Memory management efficiency decreases
    
- More likely to experience stalls during peak periods
    
    

It's a common optimization item in Redis production environments, but shouldn't be the sole root cause of this half-open connection issue.

### 4. Correct understanding

This is not a "Redis compilation/installation issue," but:

- Redis production deployment failed to complete system parameter optimization
    
- Official warnings were not addressed before going live
    
- OS and Redis parameters were not aligned
    
- Lack of production environment standards
    
    

---

## Eight. Root Cause Analysis

### 1. Direct technical cause

### The Redis host failed to complete production system parameter optimization, leading to insufficient backlog capacity

The most critical evidence is:

- Redis configuration wants backlog=511
    
- But Linux `somaxconn` is only 128
    
- Actual listening queue capacity is relatively small
    
- During peak connection establishment, it cannot handle large sudden connection volumes
    
- Ultimately causing a large number of TCP half-open connections to accumulate
    
    

---

### 2. Secondary risk factors

#### Redis production environment failed to address official warnings

##### `overcommit_memory=0`

May lead to:

- Background persistence failure
    
- Fork failure
    
- Increased Redis jitter when memory is tight
    
    

##### THP not disabled

May lead to:

- Poorer memory allocation efficiency
    
- Redis latency jitter
    
- Less stable performance under high load scenarios
    
    

These two items are not the most direct causes of this half-open connection issue, but they increase overall instability during peak periods.

---

### 3. Possible business-side issues

Although Redis/OS parameter issues are already very clear, attributing all blame to Redis is incomplete.

From an architectural perspective, the business side may also have the following issues:

#### (1) Insufficient connection reuse

During peak periods, if the business side doesn't properly use connection pools or frequently connect/disconnect, it will lead to:

- Short connection storm
    
- Sudden surge in connection establishment
    
- Redis access layer being overwhelmed
    
    

#### (2) Improper connection pool configuration

For example:

- Too small connection pool
    
- Frequent new connection creation during peak periods
    
- Unreasonable timeout and retry strategies
    
- Some instances' abnormal retry amplification issues
    
    

#### (3) Redis being frequently cross-cloud accessed

Public cloud business frequently accessing private cloud Redis inherently amplifies the following issues:

- Connection establishment latency
    
- Cross-cloud handshake cost
    
- Session pressure on boundary devices
    
- Vulnerability of connection establishment layer under high concurrency
    
    

#### (4) Business peak traffic model may be unhealthy

For example:

- Sudden reporting
    
- Concentrated writing in a certain time period
    
- Lack of peak shaving and limiting
    
- Mixed use of MQ/cache causing abnormal concentration of Redis pressure
    
    

---

## Nine. Final Root Cause Determination

### Recommended wording for the post-mortem

> The direct technical cause of this incident is that the Redis host failed to complete system kernel parameter optimization according to production standards, with `net.core.somaxconn` only at 128, causing the Redis `tcp-backlog` configuration to fail to take effect. During business peak periods, the connection establishment queue capacity was insufficient, leading to a large number of TCP half-open connections accumulating, causing congestion in the Redis access layer and business reporting delays.  
> At the same time, the host has `overcommit_memory=0` and THP not disabled, etc., production risk items, further increasing Redis's performance jitter and stability risks under high load scenarios.  
> From a system perspective, the business-side Redis access model may also have issues such as insufficient connection reuse, short connection storms, or improper connection pool configuration. In hybrid cloud cross-network access scenarios, these factors collectively amplified this incident.

- Use iperf to identify network link bandwidth bottlenecks
    
- Locate a large number of TCP half-open connections before Redis
    
- Manually clean up half-open connections
    
- Adjust operating system kernel network parameters
    
- Enhance Redis access layer's short-term capacity
    
    

### Handling Nature

These actions belong to:

- **Emergency Hemostasis**
    
- **Quick Business Recovery**
    
- **Alleviate Access Congestion**
    
    

Rather than final architectural solutions.

---

## Eleven. Subsequent Optimization Suggestions

### 1. Operating System Layer

Key optimizations:

- `net.core.somaxconn`
    
- `net.ipv4.tcp_max_syn_backlog`
    
- `net.ipv4.tcp_fin_timeout`
    
- `net.ipv4.ip_local_port_range`
    
- `net.netfilter.nf_conntrack_max` (if the link passes through conntrack/NAT)
    
- Disable THP
    
- Set `vm.overcommit_memory=1`
    

### 2. Redis Layer

Key checks:

- `tcp-backlog`
    
- `maxclients`
    
- Slow queries
    
- bigkey / hotkey
    
- Redis command execution latency
    
- Peak CPU / memory / fork status
    
- Whether Redis is bearing too many roles (cache + MQ + statistics middleware)
    

### 3. Business Application Layer

Key investigations:

- Whether connection pooling is used correctly
    
- Whether frequent connection establishment/termination exists
    
- Whether retry mechanisms are overly aggressive
    
- Whether a short connection storm forms during peak times
    
- Whether long connection reuse can be increased
    
- Whether peak traffic mitigation and rate limiting measures can be added
    
    

### 4. Architecture Layer

Key considerations:

- Whether it's suitable to continue allowing public cloud business to frequently cross-cloud access private cloud Redis
    
- Whether an access buffer layer should be added on the public cloud side
    
- Whether the core coupling of Redis in this scenario should be reduced
    
- Whether the statistical chain should be smoothed, asynchronous, or layered processed
    
    

---

## Twelve. Lessons Learned from This Incident

### Experience 1: Don't be misled by "business intuition"

The business initially suspected a network bandwidth issue, but in reality, bandwidth was not the root cause.

### Experience 2: Distinguish between "throughput issues" and "connection establishment issues" in network problems

iperf proved throughput wasn't the issue, but it couldn't confirm the connection establishment layer was fine.

### Experience 3: Don't ignore Redis official warnings

Redis startup already warned about backlog, overcommit_memory, and THP risks. These must be addressed in production deployments.

### Experience 4: Separate emergency hemostasis from root cause fixes

Cleaning half-open connections and adjusting kernel parameters are emergency measures and don't imply the architectural issue is fully resolved.

### Experience 5: In hybrid cloud architectures, frequent cross-cloud access is more sensitive to connection models

Cross-cloud links aren't impossible to use, but they're better suited for stable traffic and not for short-term massive connection establishment storms.

---

## Thirteen. TCP State Machine Knowledge Supplement (Understanding This Redis Half-Connection Fault)

In this incident, one of the most critical technical phenomena was:

- A large number of TCP half-open connections appeared before Redis
    
- Difficulty establishing connections during peak times
    
- System lag and failure reports
    
    

To explain this issue clearly, one must understand the TCP state machine, especially the **connection establishment phase** and **connection termination phase**.

### 1. What is a TCP State Machine

TCP is a connection-oriented protocol. A connection goes through a series of state changes from establishment to termination.

Common states include:

- `LISTEN`
    
- `SYN-SENT`
    
- `SYN-RECV`
    
- `ESTAB`
    
- `FIN-WAIT-1`
    
- `FIN-WAIT-2`
    
- `CLOSE-WAIT`
    
- `LAST-ACK`
    
- `TIME-WAIT`
    
- `CLOSED`
    

These states essentially reflect:

- The current stage of the connection's lifecycle
    
- Who initiated the connection
    
- Who initiated the closure
    
- Whether the handshake and termination are complete
    
    

### 2. Three-way Handshake and Key States

TCP connection establishment relies on the three-way handshake.

#### Step 1: Client sends SYN

The client initiates a connection request to the server. At this point, the client's state becomes:

- `SYN-SENT`
    

#### Step 2: Server responds with SYN+ACK

After receiving the SYN, if the server agrees to establish the connection, it replies with SYN+ACK. At this point, the server's state becomes:

- `SYN-RECV`
    

#### Step 3: Client sends ACK

After receiving the SYN+ACK, the client sends an ACK. Then:

- The client enters `ESTAB`
    
- The server, after receiving the ACK, also enters `ESTAB`
    

At this point, the TCP connection is truly established.

### 3. What is a "Half-Connection"

In production troubleshooting, the term "half-connection" typically refers to:

- The server has received the client's SYN
    
- The server has sent a SYN+ACK
    
- But the final ACK hasn't completed
    
- The connection remains in `SYN-RECV` state
    
    

So:

### The most typical state of a half-connection is:

- **Server: `SYN-RECV`**
    

This indicates the connection is only partially established, and the handshake hasn't fully completed.

### 4. Why Do We See a Large Number of Half-Connections

A large number of `SYN-RECV` typically occurs due to the following reasons:

#### (1) Clients initiate a large number of connections

For example, during peak times, a large number of clients simultaneously connect to Redis.

#### (2) The server's access capacity is insufficient

For example:

- The backlog queue is too small
    
- `somaxconn` is too low
    
- The Redis `tcp-backlog` configuration is ineffective
    
- Redis itself can't handle the load
    
- CPU is busy, there are many slow commands, and accept is delayed
    
    

#### (3) The link has packet loss, jitter, or latency

If ACKs return late, the server may remain in `SYN-RECV` for a long time.

#### (4) There is malicious SYN Flood or similar traffic patterns

A large number of SYNs come in, but ACKs aren't fully returned, causing half-connection accumulation.

### 5. How Did the TCP State Machine Manifest in This Incident

In this incident:

- About 140,000 half-connections appeared before the Redis cluster
    
- Essentially, many connections remained in `SYN-RECV`
    
- This indicates the Redis access layer couldn't complete connection establishment during peak times
    
- New requests kept coming in, while old requests hadn't completed the handshake, eventually causing congestion in the connection establishment layer
    
    

Combined with Redis logs: /think

WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

It can be inferred that:

- Redis wants a larger backlog
    
- But the system `somaxconn` is too small
    
- The server's listening queue capacity is limited
    
- During peak connection times, a large number of connections pile up in `SYN-RECV`
    
- Eventually forming a half-open connection storm
    

### 6. Relationship between LISTEN, half-open queue, and full connection queue

After the server process (e.g., Redis) starts, it will be in:

- `LISTEN`
    

This indicates the server is listening on the port, waiting for client connections.

But `LISTEN` is just the listening state. The actual problem during peak times is the two queues:

#### (1) Half-open queue (SYN Queue)

Used to store connections that have not yet completed the three-way handshake.

That is:

- The server just received a SYN
    
- Sent a SYN+ACK
    
- Still waiting for the final ACK
    

These connections typically correspond to the server's `SYN-RECV` state.

#### (2) Full connection queue (Accept Queue)

The three-way handshake has been completed, the connection is established, but the application hasn't yet `accept()` taken it.

If the application processes slowly or the queue is too small, it can also cause queuing and new connection blocking.

### 7. Why `somaxconn` and `tcp-backlog` are critical

Redis's `tcp-backlog` is the application's desired listening queue size, but it will ultimately be limited by system parameters.

If:

- Redis configures `tcp-backlog=511`
    
- Linux `net.core.somaxconn=128`
    

Then Redis's attempt to set 511 is ineffective, the system will only allow 128.

This means:

- The server's listening queue capacity is small
    
- A large number of connections can easily pile up during peak times
    
- Connection establishment capacity is insufficient
    
- A half-open connection surge is more likely to occur
    

So in this incident, `somaxconn` is a very critical direct evidence.

### 8. Other TCP states to recognize besides half-open connections

#### (1) `ESTAB`

Indicates that the connection has been established and can normally transmit data.

If Redis is working normally, most business connections should be in:

- `ESTAB`
    

#### (2) `TIME-WAIT`

Indicates the party actively closing the connection. After the connection is disconnected, it will retain the state for a period to ensure old packets do not affect new connections.

This state is common in:

- Many short connections
    
- High request volume
    
- Frequent connection establishment and closure
    

If there are many `TIME-WAIT` in the system, it usually indicates:

- Possible excessive short connections
    
- Insufficient connection reuse
    
- Unhealthy application connection model
    

In the Redis scenario, if the business doesn't properly reuse connections, you might also see many `TIME-WAIT`.

#### (3) `CLOSE-WAIT`

Indicates that the peer has closed the connection, but the local application hasn't closed it promptly.

If there are many `CLOSE-WAIT`, it often indicates:

- The application hasn't released the socket promptly
    
- There is a connection leak in the application
    
- There is an issue with the code's closing logic
    

This state is more application-related.

#### (4) `FIN-WAIT-1 / FIN-WAIT-2 / LAST-ACK`

These states are related to the four-way handshake and mainly describe the connection closure process.

In your Redis half-open connection incident, it's not the focus, but you can mention it in an interview:

> Connection establishment issues mainly focus on `SYN-SENT`, `SYN-RECV`, and `ESTAB`; connection closure issues are more about `TIME-WAIT`, `CLOSE-WAIT`, etc.

### 9. How to use TCP states during fault diagnosis

During on-site troubleshooting, the TCP state machine's most important value is:

#### Using connection states to infer problem stages

##### If there are many `SYN-RECV`

Usually indicates:

- Half-open connection buildup
    
- Connection establishment issues
    
- Backlog / server acceptance capacity / ACK return issues
    

##### If there are many `ESTAB`

Usually indicates:

- The connection has been established
    
- More likely the application is slow, Redis commands are slow, or downstream is blocked
    

##### If there are many `TIME-WAIT`

Usually indicates:

- Excessive short connections
    
- Poor connection reuse
    
- High frequency of connect / disconnect
    

##### If there are many `CLOSE-WAIT`

Usually indicates:

- The application hasn't released the connection promptly
    
- There may be code logic issues
    

### 10. How to explain TCP state machine in an interview based on this case

You can say:

> The key TCP phenomenon in this incident was a large number of half-open connections appearing before Redis, essentially many connections were stuck in the server's `SYN-RECV` state, indicating the three-way handshake wasn't completed promptly. Combined with Redis startup logs, we found that `tcp-backlog=511` was actually limited by `somaxconn=128`, so the server's connection establishment queue capacity was insufficient during peak times, leading to connection buildup. This issue isn't simply about bandwidth, but a typical TCP connection establishment congestion problem.

### 11. A simplified version for memorization

#### Focus on connection establishment phase:

- `SYN-SENT`
    
- `SYN-RECV`
    
- `ESTAB`
    

#### Focus on connection closure phase:

- `TIME-WAIT`
    
- `CLOSE-WAIT`
    

#### Core state in this incident:

- **`SYN-RECV`**
    
- Corresponding phenomenon: **Half-open connection buildup**
    

### 12. One-sentence summary

> The TCP state machine is a fundamental basis for troubleshooting network and connection issues; in this Redis incident, the large number of half-open connections essentially corresponds to the server's `SYN-RECV` state buildup, indicating the issue occurred during the connection establishment phase, not the data transmission bandwidth phase.

---

## Fourteen, Supplementary Commands for Troubleshooting TCP States

### 1. View all TCP listening states

ss -ntl

### 2. View Redis 6379 port-related connection states

ss -ant | grep 6379

### 3. Count the number of TCP states

ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c

### 4. Focus on half-open connections (SYN-RECV)

ss -ant | grep SYN-RECV | wc -l

### 5. Check for a large number of TIME-WAIT

ss -ant | grep TIME-WAIT | wc -l

### 6. View kernel backlog-related parameters

sysctl net.core.somaxconn  
sysctl net.ipv4.tcp_max_syn_backlog

### 7. Check Redis startup logs for backlog warnings

grep -i backlog /apps/redis/logs/redis.log

---

## FifteenI don't know.Interview Answer Template

### Version One: Concise Version

I encountered a Redis failure in a hybrid cloud scenario. Frontend business was deployed in public cloud, Redis and database in private cloud. During peak hours, the system became sluggish and data could not be reported. Initially, the business suspected cross-cloud bandwidth limitations. I first used iperf to test the link throughput, confirmed normal throughput, and ruled out simple bandwidth issues. Subsequently, I observed a large number of TCP half-open connections before Redis, peaking at about 140,000, indicating the issue was more related to congestion at the connection establishment layer. Later, reviewing Redis startup logs, I found that Redis's `tcp-backlog=511` actually failed to take effect because the host `somaxconn` only had 128, which perfectly matched the on-site half-open connection backlog. Ultimately, I concluded the direct cause was the Redis host not completing production kernel parameter optimization, leading to insufficient capacity for the connection establishment queue during peak hours. I first recovered the business by cleaning up half-open connections and optimizing kernel parameters, and later implemented long-term governance from the perspectives of business connection pools, Redis configuration, and hybrid cloud access models.

### Version Two: Expanded Version

I handled a typical Redis access layer failure. The business scenario was national wind power and water resources generation statistics, with frontend Web in public cloud, Redis cluster and database in private cloud, and large data reporting during peak hours. When the failure occurred, the system was clearly sluggish, and data could not be reported. The business's first reaction was insufficient bandwidth between public and private cloud. I first used iperf to test the link throughput, confirmed normal throughput, and shifted the direction from "insufficient bandwidth" to "insufficient connection establishment capacity." Continuing the investigation, I found approximately 140,000 TCP half-open connections before Redis, indicating congestion at the Redis access layer during peak hours. Later, reviewing Redis startup logs, I saw a critical warning: Redis wanted to set backlog to 511, but the system `somaxconn` only had 128, causing the listening queue capacity to be far below expectations. This aligned perfectly with the on-site phenomenon. Ultimately, I concluded the root cause was the Redis host not completing production kernel parameter optimization, leading to insufficient capacity for the connection establishment queue during peak hours; and `overcommit_memory=0` and THP not being disabled were also Redis stability risks. I also believed the business side might have connection pool or short connection storm issues, which, in a hybrid cloud scenario, amplified the failure together. At the time, I first recovered the business by cleaning up half-open connections and optimizing kernel parameters, and later pushed for long-term governance from Redis parameters, OS parameters, business connection reuse, and architectural levels.

---

## SixteenI don't know.One-sentence Summary

> This incident appeared to be a "hybrid cloud network issue" on the surface, but essentially it was insufficient connection establishment capacity at the Redis access layer; the direct cause was low kernel parameter configuration related to backlog on the host, leading to a backlog of TCP half-open connections during peak hours, while high concurrency connection establishment by business and hybrid cloud cross-network access jointly amplified the problem.