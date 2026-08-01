# 08-Linux Performance Analysis Methodology: USE, RED, and Troubleshooting Path Design

#Linux #Transport #SRE #PerformanceAnalysis #UseMethodology #RedApproach #GoldSignal. #TheMethodOfExclusion #FaultLocation. #HostBarriers

---

## Recommended Path

01-Linux Foundation and Host Operations/01-Host Troubleshooting/08-Linux Performance Analysis Methodology: USE, RED, and Troubleshooting Path Design.md

---

## One, Document Explanation

This document organizes the methodology for Linux host performance analysis, with the focus not on individual commands but rather:

- How to establish a troubleshooting path when encountering a fault
- How to transition from phenomena to metrics
- How to trace resources from metrics
- How to trace processes from resources
- How to trace services from processes
- How to trace root causes from services
- How to differentiate between resource issues, application issues, network issues, and storage issues
- How to use the USE method to analyze host resources
- How to use the RED method to analyze service requests
- How to use golden signals to design troubleshooting entry points
- How to avoid blind restarts, blind scaling, and blind blame-shifting

This document belongs to the advanced methodology section of the host troubleshooting series.

The previous 01-07 articles focus on commands and scenarios, while this article focuses on troubleshooting thinking.

The goal is:

To connect scattered commands into a troubleshooting path

→ To know what to check first when seeing a phenomenon

→ To determine the next direction based on metrics

→ To distinguish between host resource bottlenecks and business service bottlenecks

→ To form an advanced SRE commonly used troubleshooting analysis framework

---

## Two, Why a Performance Analysis Methodology is Needed

In production troubleshooting, many issues are not about not knowing commands, but about not knowing the order.

Common wrong practices:

```text
When the service is slow, restart.

CPU Kill the process when you're taller.

When the disk is full, delete the log.

The network doesn't make sense.

load When you're taller, you think CPU Not enough.

Memory free I don't think it's enough.

Once the port is clear, it's normal.

Yeah. 502 I thought... Nginx Problem
```

The problems with these practices are:

```text
Easy to destroy.

Easy to misdirect.

It's easy to cure the symptoms.

It's prone to secondary failure.

Unable to form a wrapping loop
```

A more advanced troubleshooting approach should be:

```text
Confirm the phenomenon first.

→ Reconfirm the scope of the impact

→ Reconfirm the timeline.

→ Read the indicators again.

→ Repositioning Resources

→ Repositioning process

→ Repositioning services

→ Revalidation assumptions

→ Re-improvement

→ Last reset governance
```

---

## Three, General Principles of Performance Analysis

Performance analysis is not about drawing conclusions based on a single high metric.

It is recommended to follow:

```text
First whole, then local.

First phenomena, later indicators

First Resources, After Process

Evidence, then operation.

Validation, then repair.

Restore first, then reset.
```

Troubleshooting should avoid:

```text
Read single indicators only

Just for instant data.

Just one machine.

Only service status.

Only apply log

Listen to user descriptions only

Based on experience.
```

It should combine:

```text
Operational phenomena

System indicators

Service Log

Network connectivity

Resource utilization rate

Error Rate

Delay

Timeline

Recent Changes
```

---

## Four, Four Levels of Host Performance Analysis

Linux host troubleshooting can be divided into four levels:

```text
First level: System level
→ CPU, memory, disk, network, load, process

Second floor: Service level
→ systemd, port, log, configuration, service-dependent

Third level: application layer
→ Request quantity, error rate, delay, thread, connect pool, slow query

Level 4: Architecture
→ Load balance, cache, database, message queue, storage, network links
```

Do not stay long in any single level during troubleshooting.

For example:

```text
Host CPU High
→ Could be a code application problem.

Interface Timeout
→ It's probably a slow search of the database.

Nginx 502
→ Maybe back-end service connections are full.

load High
→ Could be a disk. IO or NFS Carton.

Containers OOM
→ Maybe. cgroup limit It's too small. It doesn't have enough memory.
```

---

## Five, USE Method

---

## Scenario 1: What is the USE Method

USE is a common resource analysis method in performance analysis.

USE stands for:

```text
U = Utilization
→ Usage

S = Saturation
→ Saturation

E = Errors
→ Error
```

It is suitable for analyzing resource-related issues.

For example:

```text
CPU

Memory

Disk

Network

File Description

Number of connections

Thread pool

Number of processes

Line
```

One-sentence understanding:

```text
USE Methodology = See if one of the resources is fully spent.
```

---

## Scenario 2: Utilization (Usage Rate)

Utilization indicates how much of a resource is being used.

Common examples:

```text
CPU Usage 90%

Memory available Low

Disk Usage 95%

Disk IO util Close 100%

Net traffic is close to bandwidth limit.
```

Common commands:

```bash
top
```

```bash
free -h
```

```bash
df -h
```

```bash
iostat -x 1 5
```

```bash
sar -n DEV 1 5
```

---

## Scenario 3: Saturation (Saturation)

Saturation indicates whether a resource is experiencing queuing or waiting.

Common examples:

```text
CPU run queue High

load Longer CPU Numerical

Disk await High

vmstat Medium b High

Network Connection Line Full

Thread pool queue pile

Connect pool waiting
```

Common commands:

```bash
uptime
```

```bash
vmstat 1 5
```

```bash
iostat -x 1 5
```

```bash
ss -s
```

```bash
ss -ant
```

---

## Scenario 4: Errors (Error Rate)

Errors indicate whether exceptions occur during resource usage.

Common examples:

```text
Cybercard errors / dropped Growth

Disk I/O error

File System error

OOM Killer

TCP Repeat

Services failed

DNS Parsing failed

Connection refused / timeout
```

Common commands:

```bash
dmesg -T | tail -n 100
```

```bash
journalctl -p err
```

```bash
ip -s link
```

```bash
ss -s
```

```bash
journalctl -u Service Name -n 100
```

---

## Six, Application of the USE Method in Host Resources

---

## Scenario 5: CPU USE Analysis

CPU USE analysis:

```text
Utilization
→ CPU High usage rate

Saturation
→ CPU Run Queue

Errors
→ General CPU It's less direct. errorRead more about kernels, soft interruptions, movements.
```

Common commands:

```bash
top
```

```bash
mpstat -P ALL 1 5
```

```bash
vmstat 1 5
```

```bash
pidstat 1 5
```

Judgment:

```text
us High
→ User state program consumption CPU

sy High
→ Internal nuclear consumption CPU

si High
→ Soft break consumption CPU

r High
→ CPU Run queue pressure high

Single Nuclear High
→ Maybe a one-way bottleneck.
```

---

## Scenario 6: Memory USE Analysis

Memory USE analysis:

```text
Utilization
→ available Whether it's low or whether it's heavily used

Saturation
→ swap Whether to change pages frequently, or to have memory distribution waiting

Errors
→ Whether or not it happened OOM Killer
```

Common commands:

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
cat /proc/meminfo
```

```bash
dmesg -T | grep -i oom
```

```bash
journalctl -k | grep -i oom
```

Judgment:

```text
available Low
→ Insufficient available memory

si / so High
→ swap The change is obvious.

OOM
→ Process killed by kernel.

buff/cache High
→ Not necessarily a problem, but a combination. available Look.
```

---

## Scenario 7: Disk USE Analysis

Disk USE analysis:

```text
Utilization
→ Is the disk space full?IO util High

Saturation
→ await Is it high?IO Whether to line up

Errors
→ Is there any? I/O errorFile system error
```

Common commands:

```bash
df -h
```

```bash
df -hi
```

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

```bash
dmesg -T | grep -i error
```

Judgment:

```text
Use% High
→ Disk Space Pressure

IUse% High
→ inode Pressure

%util High
→ Disk Busy

await High
→ IO Long waiting time

dmesg Yes. I/O error
→ Possible disk or storage anomaly
```

---

## Scenario 8: Network USE Analysis

Network USE analysis:

```text
Utilization
→ Whether network traffic is close to bandwidth ceiling

Saturation
→ Connection queues, number of connections, repeat,TIME_WAIT Is it unusual?

Errors
→ errorsI don't know.droppedI don't know.retransmit Is it unusual?
```

Common commands:

```bash
sar -n DEV 1 5
```

```bash
iftop -n
```

```bash
ip -s link
```

```bash
ss -s
```

```bash
tcpdump -i any -nn
```

Judgment:

```text
rxkB/s High
→ High flow in direction

txkB/s High
→ High flow out.

dropped Growth
→ Maybe it won't work.

errors Growth
→ Possible cybercards, drivers, chain problems

SYN_RECV More
→ Could connect the queue or attack problem

TIME_WAIT More
→ Need to judge with connection mode
```

---

## Seven, RED Method

---

## Scenario 9: What is the RED Method

RED is a method for analyzing service requests.

RED stands for:

```text
R = Rate
→ Request rate

E = Errors
→ Error number or error rate

D = Duration
→ Time-consuming request
```

It is suitable for analyzing service-related issues.

For example:

```text
HTTP API

Nginx Services

Database request

RPC Call

Message Consumption

Business interface

Microservice Call Chain
```

One-sentence understanding:

```text
RED Methodology = See service requests, error rates and time-consuming
```

---

## Scenario 10: Rate (Request Rate)

Rate indicates the number of requests per unit time.

Common metrics:

```text
QPS

RPS

Request per minute

Connected per second

Number of log lines per second

Message consumption per second
```

Common troubleshooting approaches:

```bash
tail -f /var/log/nginx/access.log
```

```bash
wc -l /var/log/nginx/access.log
```

Minute-based statistics for Nginx request volume:

```bash
awk '{print $4}' /var/log/nginx/access.log | cut -c 2-17 | sort | uniq -c | sort -nr | head
```

---

## Scenario 11: Errors (Error Rate)

Errors indicate request failure situations.

Common metrics:

```text
HTTP 5xx

HTTP 4xx

Connection failed

Timeout

Abnormal Log

Database error

Service-dependent error

RPC Call Failed
```

Nginx statistics for status codes:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

Checking 5xx errors:

```bash
awk '$9 ~ /^5/ {print}' /var/log/nginx/access.log | tail -n 50
```

Checking error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

Real-time error viewing:

```bash
tail -f /var/log/nginx/error.log
```

---

## Scenario 12: Duration (Request Latency)

Duration indicates request processing time.

Common metrics:

```text
Average delay

P95

P99

Maximum time-consuming

upstream Response Time

Database query time-consuming

Interface processing time-consuming
```

If Nginx access log records request_time, you can statistics slow requests.

Example log format with request time as the last column can reference:

```bash
awk '$NF > 1 {print}' /var/log/nginx/access.log | tail -n 50
```

Explanation:

```text
The location of the field depends Nginx log_format
Confirm log format in production
```

Checking log format:

```bash
grep -n "log_format" /etc/nginx/nginx.conf
```

```bash
grep -R "log_format" /etc/nginx/
```

---

## Eight, Difference Between RED and USE

USE focuses on resources:

```text
CPU Are you full?

Do you have enough memory?

Did you get the disk? IO Wait

Did you lose your bag?

Is the connection lined up?
```

RED focuses on services:

```text
Any surges in requests?

Has the error rate increased?

Did the request take longer?
```

Relationship between the two:

```text
RED Identification of service problems

USE Identifying resource bottlenecks
```

For example:

```text
Interface P99 Delay escalation
→ RED Found Duration Unusual

Keep checking the mainframe. CPU / IO / Network
→ USE Identifying resource bottlenecks
```

Another example:

```text
CPU High usage
→ USE I've got a resource anomaly.

Keep checking for surges.
→ RED Determination of whether business flows result
```

---

## Nine, Golden Signals

---

## Scenario 13: What are Golden Signals

The four golden signals often mentioned in SRE:

```text
Latency
→ Delay

Traffic
→ Traffic

Errors
→ Error

Saturation
→ Saturation
```

This can be understood as:

```text
User Slow

There's a lot of requests.

There's not much failure.

Resources are not satisfactory
```

---

## Scenario 14: Latency (Delay)

Latency focuses on:

```text
Interface response time

Request processing time

Upstream waiting time

Database query time

Disk IO Wait

Network RTT
```

Common commands:

```bash
curl -w "time_namelookup:%{time_namelookup}\ntime_connect:%{time_connect}\ntime_starttransfer:%{time_starttransfer}\ntime_total:%{time_total}\n" -o /dev/null -s http://Destination Address
```

Field understanding:

```text
time_namelookup
→ DNS Time-consuming resolution

time_connect
→ TCP Connection time-consuming

time_starttransfer
→ First byte returns time-consuming

time_total
→ Total time-consuming
```

---

## Scenario 15: Traffic (Traffic)

```text
Request Volume

Number of connections

Network traffic

Entry traffic

Export flows

Message Volume

Database QPS
```

Common Commands:

```bash
sar -n DEV 1 5
```

```bash
iftop -n
```

```bash
ss -s
```

```bash
tail -f /var/log/nginx/access.log
```

---

## Scenario 16: Errors

Errors Focus:

```text
HTTP 5xx

HTTP 4xx

Connection failed

Timeout

Apply Abnormal

Core Error

Disk Error

DNS Error
```

Common Commands:

```bash
journalctl -p err
```

```bash
journalctl -u Service Name -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
dmesg -T | grep -i error
```

---

## Scenario 17: Saturation

Saturation Focus:

```text
CPU run queue

load average

Disk await

Disk util

Connect Queue

Thread pool queue

Memory swap

File description deplete
```

Common Commands:

```bash
uptime
```

```bash
vmstat 1 5
```

```bash
iostat -x 1 5
```

```bash
ss -s
```

```bash
ulimit -n
```

---

## Ten. Troubleshooting Path Design

---

## Scenario 18: Troubleshooting Path Design Principles

Troubleshooting paths should start from the phenomenon, not from the commands.

Recommended to ask first:

```text
What do users see?

Which services are affected?

Since when?

Are all users affected?

Does it affect just one room, one node, one interface?

Are there recent releases, changes, extensions, restarts, configuration modifications?

Which indicators are the first to be detected?
```

Then decide what to check.

---

## Scenario 19: Standard Troubleshooting Path

General Path:

```text
1. Recognition

2. Identification of impacts

3. Confirm the timeline.

4. View Recent Changes

5. View service status

6. View resource indicators

7. View Log Error

8. View Services Dependence

9. Verification assumptions

10. Implementation of repairs

11. Verify Restore

12. Record Duplicate
```

Corresponding Command Entry:

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
top
```

```bash
free -h
```

```bash
df -h
```

```bash
ss -tunlp
```

```bash
curl -I http://Destination Address
```

---

## Eleven. Common Phenomena to Troubleshooting Path

---

## Scenario 20: User Feedback "System is Slow"

Do not make direct judgments.

First decompose:

```text
Is the page slow?

Is the interface slow?

Yes. SSH Slow login?

Is the database slow?

Are all functions slow?

Are some of the users slow?

Was it some time slow?
```

Initial Commands:

```bash
uptime
```

```bash
top
```

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
iostat -x 1 5
```

```bash
sar -n DEV 1 5
```

Judgment Path:

```text
load High
→ Look. CPU / IO / D Status

CPU High
→ Find Gao. CPU Process

wa High
→ Check disk IO

available Low
→ Check memory and swap

High network traffic
→ Cha. iftop / tcpdump

Resources are normal.
→ Check application logs, databases, downstream dependencies
```

---

## Scenario 21: Interface Timeout

Troubleshooting Path:

```text
Is the client connected to the service?

→ Whether the service port is listening

→ Request to reach service

→ Is there a service error log

→ Backend Dependence Timeout

→ Whether host resources are bottlenecks

→ Whether or not the network is lost or reused
```

Commands:

```bash
curl -v http://Destination Address
```

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
ss -tunlp | grep Port
```

```bash
tcpdump -i any -nn host ClientIP and port Service Port
```

```bash
journalctl -u Service Name -n 100
```

```bash
top
```

```bash
iostat -x 1 5
```

---

## Scenario 22: Nginx 502 / 504

Troubleshooting Path:

```text
Nginx Is it normal in itself?

→ upstream Is the back end accessible?

→ Whether or not backend services are listening

→ Whether the backend is timed out

→ Whether the backend log is wrong

→ Is the backend host resource abnormal?

→ DNS or upstream Is the configuration abnormal?
```

Commands:

```bash
systemctl status nginx
```

```bash
nginx -t
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep " 502 " /var/log/nginx/access.log | tail -n 50
```

```bash
grep " 504 " /var/log/nginx/access.log | tail -n 50
```

Test Backend:

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -I http://BackendIP:Backend
```

---

## Scenario 23: High Load

Troubleshooting Path:

```text
Confirm. CPU Numerical

→ Decision load Is it really unusual?

→ Look. CPU us/sy/wa

→ Look. vmstat r/b

→ Find Gao. CPU Process

→ Cha. IO wait

→ Cha. D Status Process

→ Check Memory swap
```

Commands:

```bash
uptime
```

```bash
nproc
```

```bash
top
```

```bash
vmstat 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,user,stat,cmd | awk '$4 ~ /D/ {print}'
```

```bash
iostat -x 1 5
```

---

## Scenario 24: High CPU

Troubleshooting Path:

```text
Look at the whole thing. CPU

→ See if it's single.

→ Find Gao. CPU Process

→ Find Gao. CPU Thread

→ User, kernel, soft break

→ Combining application logs and business flows
```

Commands:

```bash
top
```

```bash
mpstat -P ALL 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
pidstat -t -p PID 1 5
```

```bash
top -H -p PID
```

---

## Scenario 25: Memory Insufficient

Troubleshooting Path:

```text
Look. available

→ Look. swap

→ Look. OOM

→ High Memory Process

→ See process memory map

→ Discovery, cache, normal business growth
```

Commands:

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
dmesg -T | grep -i oom
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
pmap -x PID | tail
```

```bash
cat /proc/meminfo
```

---

## Scenario 26: Disk Full

Troubleshooting Path:

```text
Look which mounts are full.

→ Look. inode Full

→ Look for a big directory.

→ Looking for big papers.

→ Logs, backups, databases,Docker Occupation

→ Backup or archive, then clean.
```

Commands:

```bash
df -h
```

```bash
df -hi
```

```bash
du -h --max-depth=1 / | sort -hr | head
```

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
lsof | grep deleted
```

---

## Scenario 27: Port Unreachable

Troubleshooting Path:

```text
Are we listening?

→ Is the listening address correct?

→ Whether or not the firewall of this machine is released

→ Security team clear.

→ Is the route correct?

→ Request arrival

→ Response of services
```

Commands:

```bash
ss -tunlp | grep Port
```

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
iptables -L INPUT -n -v
```

```bash
firewall-cmd --list-all
```

```bash
ip route get ObjectiveIP
```

```bash
tcpdump -i any -nn port Port
```

---

## Twelve. Analysis Method from Metrics to Root Cause

---

## Scenario 28: Metrics Can Only Indicate Direction, Not Directly Equal Root Cause

For example:

```text
CPU High
```

Possible Causes:

```text
Business flows surged

Code Dead Cycle

GC Frequent

Compress Task

Log processing

Encryption

The kernel system is on call.

Soft break high
```

Another example:

```text
Disk IO High
```

Possible Causes:

```text
Database big query

Backup Tasks

Log Brush

Mirror Pull

Batch compression

swap Page Break

Storage anomaly
```

Therefore, after metric anomalies, you still need to ask:

```text
Who did this?

Why now?

Is it related to change?

Is it traffic-related?

Is dependency related?

Can it be repeated?

Has the post-rehabilitation target been restored?
```

---

## Scenario 29: From Resources to Processes

After resource anomalies, the next step is usually to find the process.

CPU:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

Memory:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

IO:

```bash
iotop -o -P
```

Or:

```bash
pidstat -d 1 5
```

Network:

```bash
iftop -n
```

Connection:

```bash
ss -antp
```

---

## Scenario 30: From Process to Service

After finding the PID, confirm which service the process belongs to.

Commands:

```bash
ps -fp PID
```

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

```bash
ls -l /proc/PID/cwd
```

```bash
ls -l /proc/PID/exe
```

```bash
systemctl status 服务名
```

Purpose:

```text
确认进程归属

确认启动参数

确认配置文件路径

确认是否 systemd 管理

确认是否可以重启
```

---

## Scenario 31: From Service to Logs

After service anomalies, check the logs.

Commands:

```bash
journalctl -u 服务名 -n 100
```

```bash
journalctl -u 服务名 -f
```

```bash
tail -n 100 应用日志文件
```

```bash
tail -f 应用日志文件
```

Logs to Focus On:

```text
error

exception

timeout

refused

reset

oom

permission denied

no space left

too many open files

connection refused

connection timed out
```

---

## Thirteen. Timeline Thinking in Troubleshooting

---

## Scenario 32: Why Timeline is Important

Many failures are not sudden but related to some event.

Common Trigger Events:

```text
发布新版本

修改配置

重启服务

扩容缩容

数据库变更

证书更新

DNS 修改

网络策略修改

防火墙变更

磁盘扩容

流量突增

定时任务执行

备份任务启动
```

When troubleshooting, ask:

```text
问题什么时候开始？

开始前有没有变更？

异常指标从什么时候开始升高？

日志错误从什么时候出现？

是否每天固定时间发生？

是否和某个定时任务重合？
```

---

## Scenario 33: View Logs by Time

systemd Logs:

```bash
journalctl --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

Specify Service:

```bash
journalctl -u 服务名 --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

Nginx logs filtered by time need to combine with log format.

Simple view for a specific hour:

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log
```

---

## Fourteen. Contrast Thinking in Troubleshooting

---

## Scenario 34: Why Contrast is Needed

Single-point data is prone to misjudgment.

Recommend to contrast:

```text
故障机器 vs 正常机器

故障时间 vs 正常时间

故障版本 vs 上一版本

异常接口 vs 正常接口

异常节点 vs 其它节点

变更前 vs 变更后
```

Contrast Items:

```text
CPU

内存

磁盘 IO

网络连接

日志错误

配置文件

服务版本

进程参数

端口监听

依赖地址
```

---

## Scenario 35: Configuration File Contrast

Commands:

```bash
diff -u old.conf new.conf
```

Directory Contrast:

```bash
diff -ruN old_dir new_dir
```

Validate Files:

```bash
md5sum 文件名
```

```bash
sha256sum 文件名
```

---

## Fifteen. Hypothesis Verification in Troubleshooting

---

## Scenario 36: Do Not Draw Conclusions Directly

Wrong Way:

```text
接口慢
→ 肯定是数据库慢

端口不通
→ 肯定是防火墙

load 高
→ 肯定 CPU 不够

内存高
→ 肯定内存泄漏
```

Correct Way:

```text
提出假设

→ 找证据验证

→ 证据不支持就换方向

→ 证据支持再处理
```

---

## Scenario 37: Hypothesis Verification Example

Phenomenon:

```text
用户访问接口超时
```

Hypothesis 1:

```text
服务端口不通
```

Verification:

```bash
nc -zv -w 2 服务IP 服务端口
```

If the port is reachable, Hypothesis 1 is invalid.

Hypothesis 2:

```text
HTTP 请求处理慢
```

Verification:

```bash
curl -w "time_total:%{time_total}\n" -o /dev/null -s http://服务地址
```

Hypothesis 3:

```text
后端依赖超时
```

Verification:

```bash
tail -n 100 应用日志
```

```bash
grep -i timeout 应用日志
```

---

## Sixteen. Recovery Priority in Troubleshooting

---

## Scenario 38: Prioritize Recovery or Root Cause

Production failures should be distinguished:

```text
止血恢复

根因分析

长期治理
```

When the impact of the failure is severe, the priority objective is:

```text
恢复业务
```

But efforts should be made to preserve the scene before recovery.

You can first collect:

```bash
uptime
```

```bash
top -b -n 1 | head -n 30
```

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
df -h
```

```bash
ss -s
```

```bash
journalctl -u 服务名 -n 200
```

Then execute recovery actions.

---

## Scenario 39: Common Recovery Actions

Common recovery actions:

```text
回滚版本

重启异常服务

切走故障节点

扩容实例

清理磁盘

释放连接

恢复配置

切换备用服务

临时限流

禁用异常入口

扩容磁盘
```

Note:

```text
恢复动作不等于根因解决
```

A post-recovery review is still required.

---

## Seventeen. Troubleshooting Record Template

---

## Scenario 40: Troubleshooting Record Suggestions

Suggested records:

```text
故障标题：

发生时间：

发现方式：

影响范围：

用户现象：

涉及服务：

涉及主机：

最近变更：

初始判断：

排查过程：

关键证据：

临时恢复措施：

恢复时间：

根因分析：

长期整改：

负责人：

截止时间：
```

---

## Scenario 41: Command Output Preservation

During troubleshooting, you can save on-site information.

Create a directory:

```bash
mkdir -p /tmp/troubleshooting-$(date +%F-%H%M%S)
```

Save basic information:

```bash
uptime > /tmp/troubleshooting-$(date +%F-%H%M%S)/uptime.txt
```

A more practical approach is to first define variables:

```bash
TS=$(date +%F-%H%M%S)
```

```bash
mkdir -p /tmp/troubleshooting-$TS
```

```bash
uptime > /tmp/troubleshooting-$TS/uptime.txt
```

```bash
free -h > /tmp/troubleshooting-$TS/free.txt
```

```bash
df -h > /tmp/troubleshooting-$TS/df.txt
```

```bash
vmstat 1 5 > /tmp/troubleshooting-$TS/vmstat.txt
```

```bash
ss -s > /tmp/troubleshooting-$TS/ss-summary.txt
```

```bash
journalctl -p err -n 200 > /tmp/troubleshooting-$TS/journal-errors.txt
```

---

## Eighteen. Common Troubleshooting Entry Command Summary

---

## System Overview

```bash
uptime
```

```bash
top
```

```bash
free -h
```

```bash
vmstat 1 5
```

---

## CPU

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

## Memory

```bash
free -h
```

```bash
cat /proc/meminfo
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
dmesg -T | grep -i oom
```

---

## Disk

```bash
df -h
```

```bash
df -hi
```

```bash
du -h --max-depth=1 / | sort -hr | head
```

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

## Network

```bash
ip a
```

```bash
ip route
```

```bash
ss -tunlp
```

```bash
ss -s
```

```bash
sar -n DEV 1 5
```

```bash
tcpdump -i any -nn
```

---

## Services

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
journalctl -u Service Name -f
```

```bash
ss -tunlp | grep Port
```

---

## Logs

```bash
journalctl -p err
```

```bash
dmesg -T | tail -n 100
```

```bash
tail -n 100 /var/log/messages
```

```bash
tail -n 100 /var/log/syslog
```

---

## Nineteen. Production Troubleshooting Notes

---

## 1. Don't Focus on a Single Metric

For example:

```text
load High
```

You need to continue confirming:

```text
CPU High

IO wait High

D Is there a lot of status processes

swap Whether or not

Is there a storage anomaly?
```

---

## 2. Don't Restart Directly to Hide the Scene

Before restarting, at least collect:

```bash
uptime
```

```bash
top -b -n 1 | head -n 30
```

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
journalctl -u Service Name -n 200
```

---

## 3. Don't Treat Recovery as the Root Cause

For example:

```text
Restore after restart
```

Can only indicate:

```text
Restarting eased the problem.
```

Cannot indicate:

```text
Gene has been solved.
```

Still need to continue analyzing:

```text
Why is the service card dead?

Why is the memory rising?

Why the leak?

Why is the disk full?

Why rely on overtime?
```

---

## 4. Don't Analyze the Host in Isolation from Business

Normal host resources don't guarantee normal business operations.

Also need to check:

```text
Error Rate

Delay

Request Volume

Dependency status

Log anomaly

User Impact
```

---

## 5. Don't Analyze Issues in Isolation from the Timeline

Many failures are related to changes.

Must confirm:

```text
Recent Publication

Whether to reconfigure

Whether to expand

Whether to restart

Whether to clean

Is there a scheduled task?

Is there a surge in traffic?
```

---

## Twenty. One-Sentence Summary

The core of Linux performance analysis methodology is:

```text
It's a phenomenon.

Validate with indicators

Conceal by Path

With evidence.

Let's get back to business.

Rewind governance
```

The USE method is suitable for observing resources:

```text
Utilization
→ Resource utilization rate

Saturation
→ Whether resources are queued or saturated

Errors
→ Is there a mistake?
```

The RED method is suitable for observing services:

```text
Rate
→ Request Volume

Errors
→ Error Rate

Duration
→ Time-consuming request
```

Golden signals are suitable as monitoring and troubleshooting entry points:

```text
Latency
→ Delay

Traffic
→ Traffic

Errors
→ Error

Saturation
→ Saturation
```

Standard troubleshooting path:

```text
Recognition

→ Identification of impacts

→ Confirm the timeline.

→ View Recent Changes

→ View service status

→ View resource indicators

→ View Log Error

→ View Services Dependence

→ Verification assumptions

→ Implementation of repairs

→ Verify Restore

→ Record Duplicate
```

Production recommendations:

```text
Watch and then act.

We'll take the test and fix it.

We'll recover first.

Let's do it again. We'll do it later.

Don't look at single indicators.

Don't start blindly.

Don't take temporary recovery as a root cause.
```