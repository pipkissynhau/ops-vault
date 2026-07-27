# 12-Host Inspection and Fault Review: Baselines, Automated Inspections, RCA, and Governance Loop

#Linux #Ops #SRE #Host Inspection #Automated Inspection #Fault Review #RCA #Governance Loop #Baseline Management #Stability Building

---

## Recommended Reading Path

01-Linux Basics and Host Ops/01-Host Troubleshooting/12-Host Inspection and Fault Review: Baselines, Automated Inspections, RCA, and Governance Loop.md

---

## I. Document Overview

This document compiles information on Linux host inspection, automated inspections, fault review, and governance loop.

Key points include:

- Why advanced SRE is more than just on-site troubleshooting
- Host stability baselines
- Scope of host inspections
- CPU/memory/disk/network inspections
- Service status inspections
- Log keyword inspections
- Security and account inspections
- Docker/Kubernetes node inspections
- Design concepts for automated inspection scripts
- Inspection result classification
- On-site fault preservation
- RCA root cause analysis
- Fault review templates
- Improvement loop closure
- Transforming a single fault into standardized capabilities

This document serves as the concluding chapter in the host troubleshooting series.

The previous 01-11 chapters focused on troubleshooting commands, resource analysis, and advanced performance debugging. This chapter emphasizes:

```text
How to reduce the occurrence of similar issues in the future
```

The goal is to:

- Establish a host inspection baseline

- Design automated inspection scripts

- Identify potential host risks in advance

- Preserve the fault scene upon occurrence

- Conduct RCA root cause analysis

- Transform a single fault into monitoring, alerts, documentation, scripts, and processes

- Develop advanced SRE stability governance mindset

---

## II. Differences Between Advanced SRE and Regular Ops

Regular ops focus more on:

```text
How to fix problems when they occur

How to restore services when they fail

How to free up occupied disks

How to diagnose port connectivity issues

How to identify processes causing high CPU usage
```

Advanced SRE not only focuses on "fixing problems" but also on:

```text
Why the problem occurred

Whether it was detected in advance

Whether there are monitoring alerts

Whether there are standard inspection procedures

Whether automated handling is available

Whether measures were taken to prevent recurrence

Whether a review and improvement loop was established
```

In short:

```text
Regular ops address individual faults

Advanced SRE builds a stability system
```

---

## III. Overall Host Governance Approach

Host governance can be divided into four layers:

```text
First layer: Inspection
→ Identify risks in advance

Second layer: Alerts
→ Detect anomalies promptly when they occur

Third layer: Troubleshooting
→ Quickly locate and restore services during failures

Fourth layer: Review
→ Establish an improvement loop after a fault occurs
```

Corresponding relationships:

```text
Inspection
→ Regularly identify disk, memory, service, log, and security risks

Alerts
→ Real-time detect resource, service, port, and error rate anomalies

Troubleshooting
→ Quickly locate the root cause based on observed symptoms

Review
→ Transform experience into monitoring, scripts, documentation, and processes
```

---

## IV. Host Stability Baselines

---

## Scenario 1: What is a Host Stability Baseline?

A host stability baseline can be understood as:

```text
The minimum stable operating standards that a production Linux host should consistently meet
```

For example:

```text
Disk usage should not exceed 80% for an extended period

Inode usage should not exceed 80% for an extended period

CPU load should not exceed the number of CPU cores

Available memory should not be too low continuously

Swap should not frequently cause paging

Core services must remain in the active state

Critical ports must be listening

System logs should not consistently display error/fail/oom messages

Time synchronization must be functioning correctly

Important configurations must have backups

Docker/containerd data directories should not fill up the system disk
```

The significance of baselines:

```text
Turn experience into rules

Move problem detection ahead of failures

Replace manual judgment with automated checks

Transform temporary fixes into long-term governance measures
```

---

## Scenario 2: Core Objects of Host Inspection

Host inspections typically include:

```text
Basic system information

CPU and load

Memory and swap

Disk space and inodes

Disk I/O

Network status and ports

System service status

System logs

Security logins

Account permissions

Time synchronization

Scheduled tasks

Docker/containerd

Kubernetes node status

Backup tasks

Configuration changes
```

---

## V. Basic Information Inspections

---

## Scenario 3: Viewing the Host Name

```bash
hostname
```

To view complete information:

```bash
hostnamectl
```

Purpose:

```text
Confirm the current host being operated on

Verify the system version

Check the kernel version

D```markdown
Is there a large number of abnormal processes reading and writing?
Are backup tasks consuming all IO resources?
Is there excessive log output?
Are there any database IO abnormalities?
Is the IO usage in Docker/containerd directories high?
```

---

## Section 10: Network and Port Inspection

---

## Scenario 21: Checking Network Cards and IPs

```bash
ip a
```

Check routes:

```bash
ip route
```

View the default route:

```bash
ip route | grep default
```

---

## Scenario 22: Checking Port Listening

```bash
ss -tunlp
```

Check specific critical ports:

```bash
ss -tunlp | grep 22
```

```bash
ss -tunlp | grep 80
```

```bash
ss -tunlp | grep 443
```

```bash
ss -tunlp | grep 6443
```

Inspection focus:

```text
Are critical service ports being listened on?
Are they listening on 127.0.0.1 or 0.0.0.0?
Are they being occupied by abnormal processes?
Are there any ports that should not be exposed?
```

---

## Scenario 23: Checking TCP Status

```bash
ss -s
```

Count by status:

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

Inspection focus:

```text
Is there abnormal accumulation of CLOSE_WAIT states?
Are there any issues with SYN_RECV states?
Does TIME_WAIT match the business model?
Is the number of connections significantly higher than historical baselines?
```

---

## Scenario 24: Checking Network Card Errors and Packet Loss

```bash
ip -s link
```

Check a specific network card:

```bash
ip -s link show eth0
```

Focus on:

```text
RX errors
TX errors
RX dropped packets
TX dropped packets
```

---

## Section 11: Service Status Inspection

---

## Scenario 25: Checking Failed Services

```bash
systemctl --failed
```

If there are failed services, check:

```bash
systemctl status service_name
```

```bash
journalctl -u service_name -n 100
```

---

## Scenario 26: Checking Critical Service Status

Examples:

```bash
systemctl status ssh
```

```bash
systemctl status nginx
```

```bash
systemctl status docker
```

```bash
systemctl status containerd
```

```bash
systemctl status kubelet
```

Critical services vary depending on the host role:

Example for a general application host:

```text
General application host
→ nginx, app, node-exporter, filebeat

Docker host
→ docker, containerd

Kubernetes node
→ kubelet, containerd, node-exporter

Database host
→ mysql, postgresql, mongod

Monitoring host
→ prometheus, grafana, alertmanager
```

---

## Scenario 27: Checking if Services Start Automatically at Boot

```bash
systemctl is-enabled service_name
```

Examples:

```bash
systemctl is-enabled docker
```

```bash
systemctl is-enabled kubelet
```

---

## Section 12: System Log Inspection

---

## Scenario 28: Checking System Error Logs

For systemd:

```bash
journalctl -p err -n 100
```

Errors since startup:

```bash
journalctl -b -p err
```

Check errors from the last hour:

```bash
journalctl -p err --since "1 hour ago"
```

---

## Scenario 29: Checking Kernel Logs

```bash
dmesg -T | tail -n 100
```

Filter for errors:

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i fail
```

```bash
dmesg -T | grep -i oom
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i ext4
```

```bash
dmesg -T | grep -i nvme
```

```bash
dmesg -T | grep -i scsi
```

---

## Scenario 30: Traditional System Logs

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
tail -n 100 /var/log/messages
```

For Ubuntu / Debian:

```bash
tail -n 100 /var/log/syslog
```

Authentication logs:

```bash
tail -n 100 /var/log/secure## Scenario 42: Checking Container Status

```bash
docker ps
```

To view all containers:

```bash
docker ps -a
```

To view containers that exited abnormally:

```bash
docker ps -a --filter "status=exited"
```

---

## Scenario 43: Checking Docker Resource Usage

```bash
docker system df
```

For detailed information:

```bash
docker system df -v
```

To check container resources:

```bash
docker stats
```

---

## Scenario 44: Checking Docker Log Size

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

To view Docker log configuration:

```bash
docker info | grep -i "Logging Driver"
```

```bash
cat /etc/docker/daemon.json
```

---

## Scenario 45: Key Points for Docker Inspection

```text
Whether the Docker service is running normally.
Whether the Docker Root Dir is located on a data disk.
Whether any containers have exited abnormally.
Whether container logs are too large.
Whether images are accumulating.
Whether volumes have clear purposes.
Whether there are privileged containers.
Whether Docker sockets are being mounted.
Whether there are any resource limitations.
Whether log rotation is enabled.
```

---

## Section 17: Kubernetes Node Inspection

---

## Scenario 46: Checking Node Status

```bash
kubectl get nodes -o wide
```

To view node details:

```bash
kubectl describe node node-name
```

Key items to check:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
Allocated resources
Events
```

---

## Scenario 47: Checking kubelet Status

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet -n 100
```

For real-time monitoring:

```bash
journalctl -u kubelet -f
```

---

## Scenario 48: Checking Container Runtime Tools

containerd:

```bash
systemctl status containerd
```

```bash
crictl info
```

```bash
crictl ps
```

To view abnormal containers:

```bash
crictl ps -a
```

---

## Scenario 49: Checking Pod Distribution and Issues

```bash
kubectl get pod -A -o wide
```

To view non-Running Pods:

```bash
kubectl get pod -A | grep -v Running
```

To check Events:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

## Section 18: Designing Automated Inspection Scripts

---

## Scenario 50: Why Automated Inspection is Needed

Problems with manual inspections:

```text
Prone to missing items.
Depends on individual experience.
Results are not traceable.
Cannot be compared over time.
Not suitable for a large number of hosts.
Unable to identify trends.
```

Goals of automated inspection:

```text
Unified inspection items.
Unified output format.
Unified threshold settings.
Unified risk levels.
Ability to retain historical data.
Integration with alerts.
Generation of daily or weekly reports.
```

---

## Scenario 51: Principles for Designing Inspection Scripts

Inspection scripts should follow these guidelines:

```text
Prefer read-only operations.
Do not automatically delete anything.
Do not restart systems automatically.
Do not modify configurations automatically.
Avoid performing high-risk tasks.
Provide clear output.
Include threshold-based checks.
Assign risk levels.
Generate log files.
Ensure scripts can be executed repeatedly.
Allow for the addition of new inspection items.
```

Examples of risk levels:

```text
OK
→ Normal
WARN
→ Requires attention
CRITICAL
→ Needs immediate action
UNKNOWN
→ Unable to determine, requires manual verification
```

---

## Scenario 52: Example of a Simple Host Inspection Script

```bash
#!/bin/bash

echo "===== Basic Host Information ====="
hostname
date
uptime
cat /etc/os-release | head -n 3

echo
echo "===== CPU Load ====="
CPU_CORES=$(nproc)
LOAD_1=$(uptime | awk -F'load average:' '{print $2}' | awk -F',' '{print $1}' | xargs)
echo "Number of CPU cores: $CPU-CoreS"
echo "1-minute load: $LOAD_1"

echo
echo "===== Memory Usage ====="
free -h

echo
echo "===== Disk Space Usage ====="
df -h

echo
echo "===== Inode Usage ====="
df -hi

echo
echo "===== Failed Services ====="
systemctl --failed

echo
echo "===== Top CPU Processes ====="
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 10

echo
echo "===== Top Memory Processes ====="
ps -eo pid,pptar czf /tmp/incident-$TS.tar.gz -C /tmp incident-$TS

View:

```bash
ls -lh /tmp/incident-$TS.tar.gz
```

---

## Chapter 20: RCA Root Cause Analysis

---

## Scenario 61: What is RCA

RCA:

```text
Root Cause Analysis
```

That is, root cause analysis.

The goal of RCA is not to write a report, but to answer the following questions:

```text
Why did it happen?
Why wasn't it detected in advance?
Why did the impact escalate?
Why did it take so long to recover?
How can it be prevented in the future?
```

---

## Scenario 62: RCA is Not Just a Description of Symptoms

Incorrect approach:

```text
Cause: The service went down.
Action: Restart the service.
Result: Recovery.
```

This is not RCA.

A more comprehensive analysis should delve deeper into these questions:

```text
Why did the service go down?
Why didn't it restart automatically after going down?
Why didn't the monitoring system issue an alarm in time?
Why could it recover after being restarted?
Why wasn't it noticed that memory usage was continuously increasing?
Why weren't log sizes limited?
Why wasn't there any alert when the disk became full?
```

---

## Scenario 63: The Five-Question Method

Example of the five-question method:

```text
Issue: Service unavailability.

Why is the service unavailable?
→ Because the application process exited.

Why did the application process exit?
→ Because it was killed by the OOM Killer.

Why did OOM occur?
→ Because memory usage continued to increase and exceeded the limit.

Why did memory usage keep rising?
→ Because the new version of the cache had no upper limit.

Why wasn't it detected in advance?
→ Because there were no alerts for process memory trends, nor was there a post-release monitoring mechanism.
```

The ultimate root cause might be:

```text
Defective cache strategy in the new version + Lack of memory trend alerts + Insufficient post-release monitoring
```

Rather than just:

```text
The process was killed by OOM.
```

---

## Chapter 21: Fault Review Templates

---

## Scenario 64: Basic Fault Review Template

```text
# Fault Review: Title

## I. Basic Fault Information

Fault time:
Discovery time:
Recovery time:
Duration of fault:
Impact scope:
Affected users:
Involved systems:
Fault severity:
Responsible person:

## II. Fault Symptoms

Symptoms on the user side:
Symptoms in monitoring:
Symptoms in logs:
Symptoms on the host side:

## III. Timeline

Time point 1:
Time point 2:
Time point 3:
Time point 4:

## IV. Troubleshooting Steps

Step 1:
Step 2:
Step 3:
Key evidence:

## V. Root Cause Analysis

Immediate cause:
Root cause:
Precipitating factor:
Reasons for the escalation of impact:

## VI. Recovery Measures

Temporary measures:
Rollback / Restart / Scaling / Switching:
Recovery verification:

## VII. Impact Assessment

Business impact:
Data impact:
User impact:
SLA/SLO impacts:

## VIII. Identified Issues

Monitoring issues:
Alerting issues:
Process flow issues:
Capacity issues:
Change management issues:
Architecture issues:

## IX. Corrective Actions

Action item 1:
Responsible person:
Deadline:
Acceptance criteria:

Action item 2:
Responsible person:
Deadline:
Acceptance criteria:

## X. Lessons Learned

New monitoring measures to add:
New alerts to implement:
New inspection routines:
New documentation to create:
New scripts to develop:
Process optimization:
```

---

## Chapter 22: Closed-Loop Correction

---

## Scenario 65: What is a Closed-Loop Correction

A closed-loop correction means that the process doesn't end just after the review is completed.

A true closed loop requires:

```text
Corrective actions planned
Responsible persons assigned
Deadlines set
Acceptance criteria defined
Tracking of progress
Final verification
```

The status of the closed loop can be one of the following:

```text
Pending
In progress
Completed
Verified
Delayed
Canceled
```

---

## Scenario 66: Examples of Corrective Actions

### Disk Full Fault

Corrective actions:

```text
Add an alert for high disk usage in /var/log
Configure logrotate
Limit debug logs generated by applications
Configure log rotation for the Docker daemon
Move the Docker data-root directory to a dedicated data drive
```

---

### OOM Fault

Corrective actions:

```text
Implement monitoring for process memory trends
Set reasonable requests/limits for containers
Optimize JVM heap parameters
Fix the issue of unlimited cache size
Monitor memory usage curves after releases
```

---

### Port Unreachable Fault

Corrective actions:

```text
Establish a baseline for service port usage
Add```bash
tail -n 100 /var/log/messages
```

```bash
tail -n 100 /var/log/syslog
```

---

## Security

```bash
who
```

```bash
w
```

```bash
last
```

```bash
lastb
```

```bash
awk -F: '$7 !~ /nologin|false/ {print $1,$7}' /etc/passwd
```

```bash
awk -F: '$3 == 0 {print}' /etc/passwd
```

```bash
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
```

```bash
sshd -t
```

---

## Time Synchronization

```bash
date
```

```bash
timedatectl
```

```bash
systemctl status chronyd
```

```bash
chronyc sources -v
```

```bash
chronyc tracking
```

---

## Cron

```bash
crontab -l
```

```bash
ls -lah /etc/cron.d/
```

```bash
tail -n 100 /var/log/cron
```

```bash
grep CRON /var/log/syslog | tail -n 100
```

---

## Docker

```bash
systemctl status docker
```

```bash
docker version
```

```bash
docker info
```

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker system df
```

```bash
docker stats
```

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

---

## Kubernetes

```bash
kubectl get nodes -o wide
```

```bash
kubectl describe node node-name
```

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet -n 100
```

```bash
kubectl get pod -A -o wide
```

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

```bash
crictl info
```

```bash
crictl ps -a
```

---

## Incident Collection

```bash
TS=$(date +%F-%H%M%S)
```

```bash
mkdir -p /tmp/incident-$TS
```

```bash
uptime > /tmp/incident-$TS/uptime.txt
```

```bash
top -b -n 1 | head -n 80 > /tmp/incident-$TS/top.txt
```

```bash
free -h > /tmp/incident-$TS/free.txt
```

```bash
vmstat 1 5 > /tmp/incident-$TS/vmstat.txt
```

```bash
df -h > /tmp/incident-$TS/df-h.txt
```

```bash
df -hi > /tmp/incident-$TS/df-hi.txt
```

```bash
ss -s > /tmp/incident-$TS/ss-summary.txt
```

```bash
journalctl -p err -n 200 --no-pager > /tmp/incident-$TS/journal-errors.txt
```

```bash
tar czf /tmp/incident-$TS.tar.gz -C /tmp/incident-$TS
```

---

## Summary

The goal of host inspection is to:

```text
Detect issues early

Turn experience into rules

Automate manual checks

Transform one-time fixes into long-term solutions
```

Key areas for inspection include:

```text
CPU / Memory / Disk / Network

Service status

System logs

Security login practices

Time synchronization

Scheduled tasks

Docker / Kubernetes node health
```

When collecting incident data, follow these principles:

```text
Collect first

Recover later

Document evidence before acting

Stop the issue before investigating the root cause
```

RCA analysis should address:

```text
Why did it happen?

Why wasn't it detected earlier?

Why did the impact worsen?

Why was recovery slow?

How can this be prevented in the future?
```

A corrective action plan must include:

```text
Rectification items

Responsible person

Deadline

Acceptance criteria

Final verification
```

The value of an advanced SRE lies not in resolving every issue, but in:

```text
Reducing the frequency of similar issues

Detecting problems more promptly

Speeding up recovery processes

Developing a systematic approach to learning from mistakes
```