# 12-Host Inspection and Fault Review: Baseline, Automated Inspection, RCA, and Governance Loop Closure

#Linux #Transport #SRE #HostInspection #AutomaticInspection #FaultRewind #RCA #GovernanceClosedCircle #BaselineManagement #Stability-building

---

## Recommended Path

01-Linux Foundation and Host Operations/01-Host Troubleshooting/12-Host Inspection and Fault Review: Baseline, Automated Inspection, RCA, and Governance Loop Closure.md

---

## I. Document Explanation

This document organizes content related to Linux host inspection, automated inspection, fault review, and governance loop closure.

This article focuses on:

- Why advanced SREs are not just about on-site troubleshooting
- Host stability baseline
- Host inspection scope
- CPU / Memory / Disk / Network inspection
- Service status inspection
- Log keyword inspection
- Security and account inspection
- Docker / Kubernetes node inspection
- Automated inspection script design philosophy
- Inspection result grading
- Fault scene preservation
- RCA root cause analysis
- Fault review template
- Rectification loop closure
- Transforming a single fault into standardized capabilities

This document is the governance conclusion of the host troubleshooting series.

The previous 01-11 articles focus on troubleshooting commands, resource analysis, and advanced performance localization, while this article emphasizes:

```text
How to keep similar problems from happening.
```

The goal is:

- To establish a host inspection baseline
- To design automated inspection scripts
- To detect host risks in advance
- To preserve the fault scene when a fault occurs
- To complete RCA root cause analysis
- To transform a single fault into monitoring, alerts, documentation, scripts, and processes
- To possess advanced SRE stability governance thinking

---

## II. Differences Between Advanced SRE and Regular Operations

Regular operations focus more on:

```text
How do you fix it?

How do you pull the service?

How do we clear the disk when it's full?

How do we check the port?

CPU How do you find the process when you're tall?
```

Advanced SREs don't just focus on "fixing", they also focus on:

```text
Why did that happen?

Any early discoveries?

Any surveillance reports?

Any standard inspections?

Is there automated processing?

Did you prevent it from happening again?

Do you have resets and reset rings?
```

One-sentence understanding:

```text
General operating capability to solve single failures

Advanced SRE Building stability systems
```

---

## III. Overall Host Governance Approach

Host governance can be divided into four layers:

```text
First level: Inspection
→ Early identification of risks

Second floor: alarm.
→ It was discovered in time for the anomaly.

Level 3: Barriers
→ Rapid location and recovery in case of malfunction

Level 4: Rewind
→ After the failure, it's formed a whole loop.
```

Corresponding relationship:

```text
Inspection
→ Disk, memory, service, logs, security risks regularly detected

Police!
→ Real-time detection of resources, services, ports, errors

The barrier.
→ Fast-tracking root causes.

Rewind
→ Deposition of experience as surveillance, scripts, documents, processes
```

---

## IV. Host Stability Baseline

---

## Scenario 1: What is a Host Stability Baseline

A host stability baseline can be understood as:

```text
One production Linux Minimum stable operating standard that the host should meet in the long term
```

For example:

```text
Disk usage cannot be exceeded in the long term 80%

inode Use cannot be exceeded in the long term 80%

CPU load Can't be longer than CPU Numerical

Memory available Can't be too low for long.

swap Should not keep changing pages frequently

Core services must be in place active Status

Key ports must be listening.

The system log does not last. error / fail / oom

Time sync must be normal

Important configuration must have backup

Docker / containerd Data directories cannot fill system disks
```

Significance of the baseline:

```text
Turning experience into rules

Move the problem before it breaks.

Turn manual judgement into automated inspection.

Turning temporary fire relief into long-term governance
```

---

## Scenario 2: Core Objects of Host Inspection

Host inspection typically includes:

```text
System Basic Information

CPU With Load

Memory & swap

Disk Space and inode

Disk IO

Network status and ports

System service status

System Log

Secure Login

Account Permissions

Time Sync

Planned tasks

Docker / containerd

Kubernetes Node Status

Backup Tasks

Configure Changes
```

---

## V. Basic Information Inspection

---

## Scenario 3: Check Hostname

```bash
hostname
```

Check complete information:

```bash
hostnamectl
```

Purpose:

```text
Confirm Current Operator Host

Confirm System Version

Confirm kernel version

Confirm host role
```

---

## Scenario 4: Check System Version

Ubuntu / Debian:

```bash
cat /etc/os-release
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
cat /etc/redhat-release
```

General:

```bash
uname -a
```

Check kernel version:

```bash
uname -r
```

---

## Scenario 5: Check Uptime

```bash
uptime
```

Focus on:

```text
How long has the system been running?

load average Is it unusual?

Current login users
```

If the machine has restarted recently, continue to check the restart records.

---

## Scenario 6: Check Restart Records

```bash
last reboot
```

Check recent login records:

```bash
last
```

Check current logged-in users:

```bash
who
```

```bash
w
```

---

## VI. CPU and Load Inspection

---

## Scenario 7: Check Load

```bash
uptime
```

Check CPU core count:

```bash
nproc
```

Judgment:

```text
load Long term less than CPU Numerical
→ Usually.

load Long-term proximity CPU Numerical
→ Needing attention.

load Long-term significant greater than CPU Numerical
→ We need to check.
```

---

## Scenario 8: Check CPU Usage

```bash
top
```

Non-interactive one-time collection:

```bash
top -b -n 1 | head -n 30
```

Check each CPU core:

```bash
mpstat -P ALL 1 3
```

If there is no `mpstat`, install:

```bash
apt install -y sysstat
```

Or:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

---

## Scenario 9: Check High CPU Processes

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

Inspection focus:

```text
Is there an abnormal process long? High CPU

Whether there is a script or unknown process High CPU

Is there a single service full? CPU

Is there any? ksoftirqd CPU High
```

---

## VII. Memory and Swap Inspection

---

## Scenario 10: Check Memory

```bash
free -h
```

Focus on:

```text
available

buff/cache

swap
```

Judgment:

```text
available Low
→ Needing attention.

swap Continued growth in use
→ Needing attention.

si / so Longer
→ Real memory pressure is high.
```

---

## Scenario 11: Check Swap Paging

```bash
vmstat 1 5
```

Focus on:

```text
si

so
```

If it remains non-zero and high continuously, it indicates frequent paging.

---

## Scenario 12: Check High Memory Processes

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,vsz,cmd --sort=-rss | head -n 20
```

Inspection focus:

```text
Existence of service memory growth

Is there an abnormal process taking over a large amount of memory

Processes in place RSS Significant above historical baseline

Is there any? OOM Records
```

---

## Scenario 13: Check OOM Records

```bash
dmesg -T | grep -i oom
```

```bash
journalctl -k | grep -i oom
```

```bash
journalctl -k | grep -i "killed process"
```

If OOM is found, record:

```text
Time of occurrence

By kill Process

Process-based services

Cancellation OOM

Operational impact
```

---

## VIII. Disk Space and Inode Inspection

---

## Scenario 14: Check Disk Space

```bash
df -h
```

Recommended threshold:

```text
Usage < 80%
→ 正常

80% - 90%
→ 关注

90% - 95%
→ 告警

> 95%
→ High risk, need to be addressed.
```

---

## Scenario 15: Check Inode

```bash
df -hi
```

High inode usage typically indicates:

```text
Too many files.

There's plenty of post fragments.

Too many cache files

session Too many files

Provisional documents not cleared
```

---

## Scenario 16: Check Large Directories

```bash
du -h --max-depth=1 / | sort -hr | head
```

Check `/var`:

```bash
du -h --max-depth=1 /var | sort -hr | head
```

Check `/data`:

```bash
du -h --max-depth=1 /data | sort -hr | head
```

---

## Scenario 17: Find Large Files

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

Find large logs under `/var/log`:

```bash
find /var/log -type f -size +500M -exec ls -lh {} \;
```

---

## Scenario 18: Find Deleted Files Occupying Space

```bash
lsof | grep deleted
```

Suitable for troubleshooting:

```text
df Show disk full

du Big file not found
```

Common causes:

```text
The file has been deleted, but the process still holds the handle.
```

---

## IX. Disk IO Inspection

---

## Scenario 19: Check Disk IO

```bash
iostat -x 1 5
```

Focus on:

```text
%util

await

r/s

w/s

rkB/s

wkB/s
```

Judgment:

```text
%util Long-term high.
→ The disk's busy.

await Long-term high.
→ IO Long waiting time

w/s or wkB/s High
→ Writing under pressure

r/s or rkB/s High
→ Read pressure high
```

---

## Scenario 20: Check Process-Level IO

```bash
pidstat -d 1 5
```

If there is `iotop`:

```bash
iotop -o -P
```

Inspection focus:

```text
Is there an abnormal process to read and write a lot?

Is there a backup full? IO

Is there a logscreen?

Existence of database IO Unusual

Is there any? Docker / containerd Contents IO High
```

---

## X. Network and Port Inspection

---

## Scenario 21: Check Network Interface and IP

```bash
ip a
```

Check routing:

```bash
ip route
```

Check default route:

```bash
ip route | grep default
```

---

## Scenario 22: Check Port Listening

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
Whether key service ports are listening

Are you listening? 127.0.0.1

Are you listening? 0.0.0.0

Whether to be occupied by an abnormal process

Is there an unexposed port?
```

---

## Scenario 23: Check TCP Status

```bash
ss -s
```

Statistical by status:

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

Inspection focus:

```text
CLOSE_WAIT Is there an abnormal buildup?

SYN_RECV Is it unusual?

TIME_WAIT Whether or not to match business models

Whether the number of connections is significantly above the historical baseline
```

---

## Scenario 24: Check Network Interface Errors and Packet Loss

```bash
ip -s link
```

Check specific network interface:

```bash
ip -s link show eth0
```

Focus on:

```text
RX errors

TX errors

RX dropped

TX dropped
```

---

## XI. Service Status Inspection

---

## Scenario 25: Check Failed Services

```bash
systemctl --failed
``` /think

If there are failed services, check:

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

---

## Scenario 26: Check Key Service Status

Example:

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

Key services vary depending on the host role.

Example:

```text
Normal Application Host
→ nginxI don't know.appI don't know.node-exporterI don't know.filebeat

Docker Host
→ dockerI don't know.containerd

Kubernetes Nodes
→ kubeletI don't know.containerdI don't know.node-exporter

Database Host
→ mysqlI don't know.postgresqlI don't know.mongod

Monitor Host
→ prometheusI don't know.grafanaI don't know.alertmanager
```

---

## Scenario 27: Check if Services Start Automatically on Boot

```bash
systemctl is-enabled Service Name
```

Example:

```bash
systemctl is-enabled docker
```

```bash
systemctl is-enabled kubelet
```

---

## Twelve. System Log Inspection

---

## Scenario 28: Check System Error Logs

systemd:

```bash
journalctl -p err -n 100
```

Errors from this boot:

```bash
journalctl -b -p err
```

Check errors from the last hour:

```bash
journalctl -p err --since "1 hour ago"
```

---

## Scenario 29: Check Kernel Logs

```bash
dmesg -T | tail -n 100
```

Filter errors:

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

RHEL / CentOS / Rocky / AlmaLinux:

```bash
tail -n 100 /var/log/messages
```

Ubuntu / Debian:

```bash
tail -n 100 /var/log/syslog
```

Authentication logs:

```bash
tail -n 100 /var/log/secure
```

Or:

```bash
tail -n 100 /var/log/auth.log
```

---

## Thirteen. Security and Account Inspection

---

## Scenario 31: Check Login Users

```bash
who
```

```bash
w
```

Check recent logins:

```bash
last
```

Check failed login records:

```bash
lastb
```

Note:

```text
lastb Dependency /var/log/btmp
Some systems may be restricted or not enabled
```

---

## Scenario 32: Check sudo Permissions

```bash
grep -v '^#' /etc/sudoers
```

Check sudoers.d:

```bash
ls -lah /etc/sudoers.d/
```

```bash
cat /etc/sudoers.d/*
```

Production Note:

```text
Don't change anything. sudoers
Changes should be used visudo
```

---

## Scenario 33: Check System Users

```bash
cat /etc/passwd
```

Check users with login shells:

```bash
awk -F: '$7 !~ /nologin|false/ {print $1,$7}' /etc/passwd
```

Check users with UID 0:

```bash
awk -F: '$3 == 0 {print}' /etc/passwd
```

Inspection Focus:

```text
Could not close temporary folder: %s

Are there more than one? UID 0 User

Unknown account

Shouldn't exist? sudo Permissions
```

---

## Scenario 34: Check SSH Configuration

```bash
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
```

Focus on:

```text
PermitRootLogin

PasswordAuthentication

PubkeyAuthentication

Port

AllowUsers

AllowGroups
```

Check after modification:

```bash
sshd -t
```

Reload:

```bash
systemctl reload sshd
```

Or:

```bash
systemctl reload ssh
```

---

## Fourteen. Time Synchronization Inspection

---

## Scenario 35: Check System Time and Timezone

```bash
date
```

```bash
timedatectl
```

Focus on:

```text
Local time

Time zone

System clock synchronized

NTP service
```

---

## Scenario 36: Check chrony

```bash
systemctl status chronyd
```

Check synchronization sources:

```bash
chronyc sources -v
```

Check synchronization status:

```bash
chronyc tracking
```

On Ubuntu, it may be:

```bash
systemctl status systemd-timesyncd
```

---

## Scenario 37: Impact of Time Desynchronization

Time desynchronization may cause:

```text
Certificate Validation Failed

Logs are out of time.

Distribution system anomaly

Database master copying anomaly

Kubernetes Certificate or certificate Token Problem

Prometheus Indicator time anomaly

Audit difficulties
```

---

## Fifteen. Scheduled Task Inspection

---

## Scenario 38: Check Current User's crontab

```bash
crontab -l
```

Check for a specific user:

```bash
crontab -l -u Username
```

---

## Scenario 39: Check System cron Directory

```bash
ls -lah /etc/cron.d/
```

```bash
ls -lah /etc/cron.daily/
```

```bash
ls -lah /etc/cron.hourly/
```

```bash
ls -lah /etc/cron.weekly/
```

```bash
ls -lah /etc/cron.monthly/
```

---

## Scenario 40: Check cron Logs

RHEL-based systems:

```bash
tail -n 100 /var/log/cron
```

Ubuntu / Debian:

```bash
grep CRON /var/log/syslog | tail -n 100
```

Inspection Focus:

```text
Is there an abnormal timing mission?

Was there a failed mission?

Is there a big task at the peak?

Is there a risk that the script will be deleted by error?

Could not close temporary folder: %s
```

---

## Sixteen. Docker Host Inspection

---

## Scenario 41: Check Docker Status

```bash
systemctl status docker
```

```bash
docker version
```

```bash
docker info
```

---

## Scenario 42: Check Container Status

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

Check containers that exited abnormally:

```bash
docker ps -a --filter "status=exited"
```

---

## Scenario 43: Check Docker Resource Usage

```bash
docker system df
```

Detailed view:

```bash
docker system df -v
```

Check container resources:

```bash
docker stats
```

---

## Scenario 44: Check Docker Log Size

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

Check Docker log configuration:

```bash
docker info | grep -i "Logging Driver"
```

```bash
cat /etc/docker/daemon.json
```

---

## Scenario 45: Docker Inspection Focus Points

```text
Docker Is service normal?

Docker Root Dir Whether on the data disc

Is the container abnormally out?

Is the container log too big?

Whether mirrors accumulate

volume Clarity of use

Existence privileged Containers

Whether to mount Docker socket

Resource constraints

Is there a log rotation?
```

---

## Seventeen. Kubernetes Node Inspection

---

## Scenario 46: Check Node Status

```bash
kubectl get nodes -o wide
```

Check node details:

```bash
kubectl describe node Node Name
```

Focus on:

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

## Scenario 47: Check kubelet Status

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet -n 100
```

Real-time view:

```bash
journalctl -u kubelet -f
```

---

## Scenario 48: Check Container Runtime

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

Check abnormal containers:

```bash
crictl ps -a
```

---

## Scenario 49: Check Pod Distribution and Abnormalities

```bash
kubectl get pod -A -o wide
```

Check non-Running Pods:

```bash
kubectl get pod -A | grep -v Running
```

Check Events:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

## Eighteen. Automation Inspection Script Design

---

## Scenario 50: Why Need Automated Inspection

Issues with manual inspection:

```text
Easy to miss.

Relying on personal experience

The result is untraceable.

No long-term comparison

Not suitable for large hosts

No trend.
```

Goals of automated inspection:

```text
Harmonization of Inspection Items

Unified Output Format

Harmonized thresholds

Harmonization of risk ratings

Saveable History

Access to the alarm.

Could be a daily or a weekly.
```

---

## Scenario 51: Inspection Script Design Principles

Inspection scripts should follow:

```text
Read-only priority

Do not automatically delete

Do not restart automatically

Do not automatically modify configuration

High-risk operations not implemented

Output Clear

There's a threshold.

Risk level

Log file available

Duplicable

Expansible check items
```

Risk level example:

```text
OK
→ Normal

WARN
→ Needing attention.

CRITICAL
→ It needs to be handled quickly.

UNKNOWN
→ I can't judge. I need manual confirmation.
```

---

## Scenario 52: Simple Host Inspection Script Example

```bash
#!/bin/bash

echo "===== Host Basic Info ====="
hostname
date
uptime
cat /etc/os-release | head -n 3

echo
echo "===== CPU Load ====="
CPU_CORES=$(nproc)
LOAD_1=$(uptime | awk -F'load average:' '{print $2}' | awk -F',' '{print $1}' | xargs)
echo "CPU cores: $CPU_CORES"
echo "Load 1min: $LOAD_1"

echo
echo "===== Memory ====="
free -h

echo
echo "===== Disk Usage ====="
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
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 10

echo
echo "===== Listening Ports ====="
ss -tunlp

echo
echo "===== Recent System Errors ====="
journalctl -p err -n 50 --no-pager
```

---

## Scenario 53: Save Inspection Results

```bash
mkdir -p /var/log/host-check
```

```bash
bash host-check.sh > /var/log/host-check/host-check-$(date +%F-%H%M%S).log 2>&1
```

View:

```bash
ls -lh /var/log/host-check/
```

---

## Scenario 54: Schedule Inspection Execution

Edit crontab:

```bash
crontab -e
```

Example: Run daily at 9 AM:

```cron
0 9 * * * /bin/bash /opt/scripts/host-check.sh > /var/log/host-check/host-check-$(date +\%F-\%H\%M\%S).log 2>&1
```

Note:

```text
crontab Medium date Yes. % Requires conversion to \%
```

---

## Nineteen. Fault Scene Preservation

---

## Scenario 55: Why Preserve the Scene

When a fault occurs, directly restarting or cleaning up may lead to:

```text
Key log lost

Process site lost

Connection lost

Loss of resource status

Can not open message

Can't prove the cause.

The same kind of problems happen again.
```

Therefore, in production faults, it's better to collect the scene first before performing recovery actions.

---

## Scenario 56: Fault Scene Collection Directory

```bash
TS=$(date +%F-%H%M%S)
```

```bash
mkdir -p /tmp/incident-$TS
```

---

## Scenario 57: Collect System Status

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

---

## Scenario 58: Collection Process and Port

```bash
ps -ef > /tmp/incident-$TS/ps-ef.txt
```

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 30 > /tmp/incident-$TS/top-cpu-process.txt
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 30 > /tmp/incident-$TS/top-mem-process.txt
```

```bash
ss -s > /tmp/incident-$TS/ss-summary.txt
```

```bash
ss -ant > /tmp/incident-$TS/ss-ant.txt
```

```bash
ss -tunlp > /tmp/incident-$TS/ss-tunlp.txt
```

---

## Scenario 59: Collection Logs

```bash
journalctl -p err -n 200 --no-pager > /tmp/incident-$TS/journal-errors.txt
```

```bash
dmesg -T | tail -n 200 > /tmp/incident-$TS/dmesg-tail.txt
```

Specify Service:

```bash
journalctl -u Service Name -n 200 --no-pager > /tmp/incident-$TS/service-journal.txt
```

---

## Scenario 60: On-Site Packaging

```bash
tar czf /tmp/incident-$TS.tar.gz -C /tmp incident-$TS
```

View:

```bash
ls -lh /tmp/incident-$TS.tar.gz
```

---

## Twenty: RCA Root Cause Analysis

---

## Scenario 61: What is RCA

RCA:

```text
Root Cause Analysis
```

Which is Root Cause Analysis.

The goal of RCA is not to write a report, but to answer:

```text
What the hell happened?

Why didn't you notice?

Why is the impact widening?

Why did it take so long to recover?

How to avoid it later?
```

---

## Scenario 62: RCA is Not Equal to Phenomenon Description

Incorrect Writing:

```text
Reason: Service hung up.
Processing: restarting services
Result: Recovery
```

This is not RCA.

A more complete analysis should continue to ask:

```text
Why does the service hang up?

Why isn't the service hung up?

Why didn't they call the police in time?

Why can you recover when you restart?

Why didn't you see the memory rising?

Why is there no limit to the size of the log?

Why didn't you call the cops before the disk was full?
```

---

## Scenario 63: Five Whys Method

Five Whys Example:

```text
Question: Unavailable services

Why isn't the service available?
→ Because application exits

Why did the application process exit?
→ Because of being... OOM Killer Kill!

Why? OOMWhat?
→ Because memory continues to grow and exceeds limits

Why does memory continue to grow?
→ Because the new version of the cache has no ceiling

Why didn't you notice?
→ Because there are no process memory trends alerts and no post release observation mechanism
```

The final root cause may be:

```text
New Cache Policy Deficiencies + Lack of memory trend alerts + Inadequate observation
```

Rather than just:

```text
Process is being OOM
```

---

## Twenty-One: Fault Review Template

---

## Scenario 64: Basic Fault Review Template

```text
# Fragmented Duplicate: Title

## I. Basic information on failures

Other Organiser
Discovery time:
Recovery time:
Duration of failure:
Scope of impact:
Other Organiser
Involving systems:
Fault level:
Responsible:

## II. FUNCTIONS

User side phenomena:
Surveillance side phenomena:
Logside phenomenon:
Host side phenomena:

## Timeline

Time 1:
Time 2:
Time 3:
Time 4:

## IV. Queries

Step 1:
Step 2:
Step three:
Key evidence:

## V. GENDER ANALYSIS

Direct causes:
Root causes:
Incentives:
Reasons for expansion:

## VI. RESTRUCTION MEASURES

Interim measures:
Roll back / Restart / Expansion / Switch:
Restore authentication:

## Impact assessment

Operational impact:
Data impact:
User impact:
SLA / SLO Impact:

## VIII. Exposure

Surveillance:
Question of alarm:
Process issues:
Capacity issues:
Change issues:
Structural issues:

## IX. CORRECTIONS

Reorganization 1:
Responsible:
Other Organiser
Acceptance criteria:

Reorganization 2:
Responsible:
Other Organiser
Acceptance criteria:

## X. EXPERIENCE DEEPING

Other Organiser
Add a new alarm:
Other Organiser
New document:
Add script:
Process optimization:
```

---

## Twenty-Two: Rectification Closure Loop

---

## Scenario 65: What is Rectification Closure Loop

Rectification closure loop is not "ending after writing the review report".

True closure requires:

```text
There's a reorganization.

There's someone in charge.

There are deadlines.

There are acceptance standards.

Tracking status

Got Final Authentication
```

Closure status can be divided into:

```text
Pending

Processing

Completed

Authenticated

Extension

Cancel
```

---

## Scenario 66: Rectification Item Examples

### Disk Full Fault

Rectification Item:

```text
Yes /var/log Increase disk usage alarm

Configure logrotate

Limit Application debug Log

Docker daemon Configure log rotations

Will Docker data-root Move to Data Disk
```

---

### OOM Fault

Rectification Item:

```text
Increase process memory trend monitoring

Rationalize packagings requests / limits

Optimization JVM Stack parameters

Recovery of caches without ceiling

Watch memory curve after release
```

---

### Port Unreachable Fault

Rectification Item:

```text
Establish service port baseline

Increase port survival detection

Additional approval pending change in firewall rules

Security team rules incorporated into configuration management

Increase tcpdump Convergence SOP
```

---

### Conntrack Table Full Fault

Rectification Item:

```text
Increase nf_conntrack_count Usage monitoring

Optimizing short-link and retry mechanisms

Adjustment nf_conntrack_max

Embracing gateway nodes

Check out the source. IP
```

---

## Twenty-Three: From Fault to Knowledge Base

---

## Scenario 67: What toDeposition After a Fault

After a fault, at least the following should beDeposition:

```text
A duplicate report.

A barrier. SOP

One or more surveillance alarms.

A check script or check item

A risk baseline

A configuration code

An emergency response process
```

---

## Scenario 68:Deposition Example

If a disk full occurs:

```text
New document:
Disk space is full of checks. SOP

Other Organiser
df -h / df -hi Day inspection

Add a new alarm:
Disk Usage > 85% warning
Disk Usage > 90% critical
inode Usage > 85% warning

New Norms:
Application log must access logrotate
Docker Log must be configured max-size / max-file
```

If an OOM occurs:

```text
New document:
OOMKilled Check. SOP

Other Organiser
Process RSS
Containers memory usage / limit
Node MemoryPressure

New Norms:
All Pod Must Configure requests / limits
JVM Service must set stack parameters by container restriction
```

---

## Twenty-Four: Production Inspection Checklist

---

## 1. Basic Information

```text
Is the hostname correct?

System version in compliance with baseline

kernel version in compliance with baseline

Is the system running time abnormal?

Is there any recent unusual restart?

Time Synchronization Normal
```

---

## 2. CPU and Load

```text
load Is it too long?

CPU Is the usage abnormal?

Is it one-core full?

Is it abnormally high? CPU Process

Whether soft breaks are excessive

Existence of virtualization st Too high.
```

---

## 3. Memory

```text
available Is it too low?

swap Continued use

si / so Maintained high

Is there any? OOM Records

Process memory growth

Whether or not containers occur OOMKilled
```

---

## 4. Disk

```text
Whether disk usage exceeds the threshold

inode Is it near depletion?

Is there a big log file?

Is there any? deleted File occupation

Docker / containerd Is the data directory too big?

Whether there is a disk or file system error
```

---

## 5. Network

```text
Is the key card or not? UP

Default route correct

Whether key ports are listening

Whether or not to listen at the wrong address

TCP Is the state abnormal?

Is there a lot of them? CLOSE_WAIT

Is there a card? dropped / errors

conntrack Close to ceiling
```

---

## 6. Services

```text
systemctl --failed Is it empty?

Whether critical services are available active

Whether critical services are available enabled

Is there a service log? error

Whether services are restarted frequently

Is the port meeting expectations?
```

---

## 7. Security

```text
Is there an unusual login?

Any anomalies? sudo Permissions

Any anomalies? UID 0 User

SSH The configuration meets the baseline

Unknown User

Is there a risk of weakness?
```

---

## 8. Docker / Kubernetes

```text
Docker / containerd Is it normal?

Is the container abnormally out?

Is the container log too big?

Whether mirrors accumulate

volume Whether there is a clear attribution

kubelet Is it normal?

Whether Node Ready

Is there any? DiskPressure / MemoryPressure

Pod Is there a lot of anomalies?
```

---

## Twenty-Five: Summary of Common Commands

---

## Basic Information

```bash
hostname
```

```bash
hostnamectl
```

```bash
cat /etc/os-release
```

```bash
uname -a
```

```bash
uname -r
```

```bash
uptime
```

```bash
last reboot
```

---

## CPU / Memory

```bash
nproc
```

```bash
top -b -n 1 | head -n 30
```

```bash
mpstat -P ALL 1 3
```

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,vsz,cmd --sort=-rss | head -n 20
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
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
lsof | grep deleted
```

```bash
iostat -x 1 5
```

```bash
pidstat -d 1 5
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
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

```bash
ip -s link
```

```bash
sar -n DEV 1 5
```

---

## Services

```bash
systemctl --failed
```

```bash
systemctl status Service Name
```

```bash
systemctl is-enabled Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
journalctl -p err -n 100
```

---

## Logs

```bash
journalctl -b -p err
```

```bash
journalctl --since "1 hour ago"
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

## cron

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
kubectl describe node Node Name
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

## Fault Scene Collection

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
tar czf /tmp/incident-$TS.tar.gz -C /tmp incident-$TS
```

---

## Twenty-Six: One-Sentence Summary

The goal of host inspection is:

```text
We're gonna get ahead of the trouble.

Turning experience into rules

Make manual checks automated.

Turning a single barrier into long-term governance
```

Inspection Focus:

```text
CPU / Memory / Disk / Network

Service Status

System Log

Secure Login

Time Sync

Planned tasks

Docker / Kubernetes Node Status
```

Fault Scene Retention Principle:

```text
Collect first.

Recovery after

Evidence first.

Post Operation

Stop the bleeding first.

Postgen
```

RCA Root Cause Analysis should answer:

```text
Why?

Why didn't you notice?

Why is the impact widening?

Why slow?

How do you avoid it?
```

Rectification Closure Loop must include:

```text
Reorganization

Head

Deadline

Acceptance and inspection standards

Final Authentication
```

The value of advanced SRE is not to save the fire every time, but to:

```text
It keeps the same kind of trouble getting smaller.

It's getting worse and faster.

Make recovery quick.

Let the experience sink into a system.
```