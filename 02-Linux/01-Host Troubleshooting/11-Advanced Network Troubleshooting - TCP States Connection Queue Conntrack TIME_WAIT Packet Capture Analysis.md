# 11-Advanced Network Troubleshooting: TCP States, Connection Queues, conntrack, TIME_WAIT, and Packet Capture Analysis

#Linux #Transport #SRE #NetworkChecking #TCP #ConnectQueue #conntrack #TIME_WAIT #CLOSE_WAIT #SYN_RECV #tcpdump #MTU #PerformanceAnalysis

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/11-Advanced Network Troubleshooting: TCP States, Connection Queues, conntrack, TIME_WAIT, and Packet Capture Analysis.md

---

## One, Document Description

This document organizes advanced network troubleshooting methods for Linux hosts, focusing not only on determining if ports are reachable but also further analyzing:

- TCP Connection States
- `SYN_SENT`
- `SYN_RECV`
- `ESTABLISHED`
- `TIME_WAIT`
- `CLOSE_WAIT`
- Half-Connection Queue
- Full-Connection Queue
- Listen Backlog
- `somaxconn`
- `tcp_max_syn_backlog`
- conntrack Table
- nf_conntrack Table Full
- Local Temporary Port Exhaustion
- TCP Retransmission
- TCP Three-Way Handshake Packet Capture
- TCP Four-Way Fin Packet Capture
- MTU Issues
- Network Packet Loss Localization
- Deep Troubleshooting for Service Port Listening Normality but Business Unavailability

This document is part of the advanced network performance analysis section in the host troubleshooting series.

The previous 06th article has already organized network connectivity, port, routing, and traffic troubleshooting. This article continues to delve into TCP, the kernel network stack, and packet capture analysis.

The goal is:

Can understand common TCP states

→ Can determine at which stage the connection is stuck

→ Can troubleshoot full connection queue exhaustion

→ Can determine if TIME_WAIT / CLOSE_WAIT is abnormal

→ Can troubleshoot network anomalies caused by conntrack table fullness

→ Can preliminarily determine local port exhaustion

→ Can use tcpdump to verify three-way handshake, retransmission, packet loss, and MTU issues

→ Can have advanced SRE interview network deep troubleshooting thinking

---

## Two, Advanced Network Troubleshooting Overall Approach

Ordinary network troubleshooting focuses on:

```text
IP It doesn't make sense.

There's no port.

Services listening

Is the route correct?

Is the firewall clear?
```

Advanced network troubleshooting also needs to continue checking:

```text
Where's the connection card? TCP Status

Whether the connect queue is full

Is it a lot? TIME_WAIT

Is it a lot? CLOSE_WAIT

Is it a lot? SYN_RECV

Whether or not it happened TCP Repeat

conntrack Are the tables full?

Local temporary ports run out

Existence MTU Problem

Is there a card? dropped / errors

Whether the request package has arrived

Response package returned

Is three handshakes complete?

Is it normal to wave four times?
```

Recommended troubleshooting path:

```text
ss -s
→ Look. TCP Connect to Whole State

ss -ant
→ Look at the specifics. TCP Status Distribution

ss -lnt
→ Watch listening ports and queues

netstat -s / nstat
→ Look. TCP/IP Protocol Stack Error and Repeat

conntrack / nf_conntrack
→ Look at the connection trail.

ip -s link
→ Look at the bugs and drops.

sar -n TCP,ETCP,DEV,SOCK
→ Look. TCP, bugs, webcards and socket Statistics

tcpdump
→ Grab a bag to verify the request, respond to it, repeat it, shake hands,MTU
```

---

## Three, TCP State Basics

---

## Scenario 1: View TCP Connection State Statistics

### Command

```bash
ss -s
```

### Purpose

View overall socket statistics.

Focus on:

```text
TCP
ESTAB
TIME-WAIT
CLOSE-WAIT
SYN-SENT
SYN-RECV
```

Suitable for quick judgment:

```text
Is the current number of connections abnormal?

TIME_WAIT Is it a lot?

CLOSE_WAIT Whether to stack

SYN_RECV Is it unusual?

Whether large numbers of connections are abnormal
```

---

## Scenario 2: View All TCP Connections

### Command

```bash
ss -ant
```

### Parameter Description

```text
-a
→ Show all connections

-n
→ Number shows, do not parse domain names and ports First Name

-t
→ TCP
```

---

## Scenario 3: Count TCP Connections by State

### Command

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

### Purpose

Count the number of TCP connections in different states.

Common output:

```text
ESTAB 120
TIME-WAIT 500
CLOSE-WAIT 30
SYN-SENT 5
SYN-RECV 20
LISTEN 15
```

---

## Scenario 4: View Connections in Specific States

View `ESTABLISHED`:

```bash
ss -ant state established
```

View `TIME-WAIT`:

```bash
ss -ant state time-wait
```

View `CLOSE-WAIT`:

```bash
ss -ant state close-wait
```

View `SYN-SENT`:

```bash
ss -ant state syn-sent
```

View `SYN-RECV`:

```bash
ss -ant state syn-recv
```

---

## Four, Understanding Common TCP States

---

## Scenario 5: LISTEN

```text
LISTEN
→ Service is listening to port, waiting for client connection
```

View listening ports:

```bash
ss -lntp
```

Focus on:

```text
Local Address:Port

Recv-Q

Send-Q

Process
```

---

## Scenario 6: SYN_SENT

```text
SYN_SENT
→ This machine has been sent as client SYNWaiting for peer SYN-ACK
```

Common causes:

```text
Target port is out.

Target host not responding

Firewalls in the middle.

Security team not released.

Route abnormal.

Connect Queue to End

Network Throw
```

View:

```bash
ss -ant state syn-sent
```

Further verification:

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
tcpdump -i any -nn host ObjectiveIP and port Port
```

---

## Scenario 7: SYN_RECV

```text
SYN_RECV
→ This is the client. SYNOther Organiser SYN-ACKWaiting for client ACK
```

Common causes:

```text
The client didn't make three handshakes.

Network Throw

Client side anomaly

SYN flood

Half-connected queue is under pressure.

Security equipment abnormal.

Backpacking path abnormal.
```

View:

```bash
ss -ant state syn-recv
```

Count:

```bash
ss -ant state syn-recv | wc -l
```

Packet capture:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'
```

---

## Scenario 8: ESTABLISHED

```text
ESTABLISHED
→ TCP Connection established
```

View:

```bash
ss -ant state established
```

View specific port:

```bash
ss -ant state established '( sport = :8080 or dport = :8080 )'
```

Explanation:

```text
ESTABLISHED It's not necessarily unusual.
To combine business connection models,QPSNumber of connective pools, long connections
```

---

## Scenario 9: TIME_WAIT

```text
TIME_WAIT
→ Enter the one who has actively closed the connection TIME_WAIT
```

Common in:

```text
Short connections.

Service calls frequently downstream as client

Nginx / API Gateway connects backends frequently

Database connection not used

HTTP keepalive Not configured

Numerous health examinations

Pressure scene
```

View:

```bash
ss -ant state time-wait
```

Count:

```bash
ss -ant state time-wait | wc -l
```

By remote endpoint:

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

---

## Scenario 10: CLOSE_WAIT

```text
CLOSE_WAIT
→ The interface has been closed to the end and the current application has not yet been closed socket
```

This is usually more concerning.

Common causes:

```text
Apply not correctly closed connection

Connection leak

Code not available close socket

HTTP Client connection not released

Database connection not released

Applying a thread is stuck and the connection cannot be closed
```

View:

```bash
ss -ant state close-wait
```

View corresponding process:

```bash
ss -antp state close-wait
```

Count:

```bash
ss -antp state close-wait | wc -l
```

---

## Five, Deep Understanding of TIME_WAIT

---

## Scenario 11: Is a High Number of TIME_WAIT Definitely a Problem

Conclusion:

```text
TIME_WAIT It's not necessarily a problem.
```

Need to combine:

```text
Whether to influence the creation of new connections

Whether local ports are depleted

Whether to cause connection failure

Whether to match short connection traffic

Is there an abnormal increase?

Whether it happened on a client role machine
```

Generally:

```text
Services as active closure parties
→ It's easier to show up. TIME_WAIT

A lot of short connections.
→ It's easier to show up. TIME_WAIT
```

---

## Scenario 12: Common Troubleshooting Commands for TIME_WAIT

View count:

```bash
ss -ant state time-wait | wc -l
```

By target IP:

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

By target port:

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 13: Optimization Directions for High TIME_WAIT

Common optimization directions:

```text
Enable HTTP keepalive

Connect pool reuse

Reduce Short Connections

Rationalally configure client timeout

Reduced frequency of meaningless health examinations

Check if there's an abnormal retest.

Confirm whether the connection is shut down frequently downstream.
```

Do not recommend changing kernel parameters immediately.

In production, should first confirm:

```text
What kind of connection is generated? TIME_WAIT

Which machine actively shut down the connection?

Whether due to short business connection

Did it really cause a malfunction?
```

---

## Six, Deep Understanding of CLOSE_WAIT

---

## Scenario 14: Why is CLOSE_WAIT More Dangerous

`CLOSE_WAIT` indicates:

```text
The end has been set off.

Here. TCP Inn has been received. FIN

But not yet. close socket
```

So the problem is usually in the local application layer.

A large number of `CLOSE_WAIT` often indicates:

```text
Apply connection leak

Code did not close the connection

Thread's stuck.

Connection pool management anomaly

After closing the connection upstream, no resources were released.
```

---

## Scenario 15: Troubleshooting Commands for CLOSE_WAIT

View:

```bash
ss -antp state close-wait
```

Count processes:

```bash
ss -antp state close-wait | grep users | awk -F'users:' '{print $2}' | sort | uniq -c | sort -nr | head
```

View target:

```bash
ss -ant state close-wait | awk 'NR>1 {print $5}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 16: Handling Directions for CLOSE_WAIT

Handling directions:

```text
Confirm which process holds CLOSE_WAIT

View Application Log

Check connect pool configuration

Inspection HTTP Client Closes response body

Check for release of database connection

Check if the thread is stuck.

Restart the bleeding if necessary.

Follow-up repair code or connect pool configuration
```

View process:

```bash
ps -fp PID
```

View open file count:

```bash
ls /proc/PID/fd | wc -l
```

View connections:

```bash
lsof -p PID | grep TCP | head
```

---

## Seven, Deep Troubleshooting of SYN_SENT and SYN_RECV

---

## Scenario 17: Large Number of SYN_SENT

A large number of `SYN_SENT` usually indicates that the local machine as a client is failing to connect to the remote endpoint.

Troubleshoot:

```bash
ss -ant state syn-sent
```

By target statistics:

```bash
ss -ant state syn-sent | awk 'NR>1 {print $5}' | sort | uniq -c | sort -nr | head
```

Test port:

```bash
nc -zv -w 2 ObjectiveIP Port
```

Packet capture:

```bash
tcpdump -i any -nn host ObjectiveIP and port Port
```

Determine:

```text
Only SYNNothing. SYN-ACK
→ No response to the end or discard in the middle

Yes. SYN-ACKBut it didn't come back. ACK
→ We need to continue to confirm the location of the inn, firewall or grab bag.
```

---

## Scenario 18: Large Number of SYN_RECV

A large number of `SYN_RECV` usually indicates that the local machine as a server is waiting for the client to complete the three-way handshake.

Troubleshoot: /think

```bash
ss -ant state syn-recv
```

Statistics Source:

```bash
ss -ant state syn-recv | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

Viewing Queues and Kernel Statistics:

```bash
netstat -s | grep -i listen
```

```bash
nstat | grep -i Tcp
```

Packet Capture:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'
```

Common Causes:

```text
SYN flood

Client network anomaly

Backpacking path abnormal.

Security equipment discarded

Half-connected queue is under pressure.

Service Load High
```

---

## VIII. Connection Queue and Backlog

---

## Scenario 19: What is a Connection Queue

During TCP server connection establishment, two queues are typically involved:

```text
Semiconnect Queue
→ SYN queue
→ Store it. SYNResponse SYN-ACKWaiting for the end ACK Other Organiser

Connect Queue
→ accept queue
→ Three handshakes, ready for application. accept Other Organiser
```

If the queue is full, the following may occur:

```text
Client connection timed out

SYN_RECV Increase

Connection abandoned

Nginx / Application of occasional connection failed

No connection to the peak interface.
```

---

## Scenario 20: Viewing Listening Port Queue

### Command

```bash
ss -lnt
```

Focus on the following in the sample output:

```text
Recv-Q
Send-Q
Local Address:Port
```

For `LISTEN` state:

```text
Recv-Q
→ Organisation accept Number of connections

Send-Q
→ listen backlog Upper limit
```

Viewing Process:

```bash
ss -lntp
```

---

## Scenario 21: Viewing Specified Port Listening Queue

```bash
ss -lntp | grep 8080
```

If you see:

```text
Recv-Q Close Send-Q
```

It may indicate:

```text
Apply accept Untimely

All lines close to full.

We can't handle the line of business.

Apply stuck or CPU Busy
```

---

## Scenario 22: Viewing somaxconn

```bash
cat /proc/sys/net/core/somaxconn
```

Explanation:

```text
somaxconn
→ System Layer listen backlog Upper limit
```

Temporary Adjustment:

```bash
sysctl -w net.core.somaxconn=4096
```

Permanent Configuration:

```bash
vi /etc/sysctl.conf
```

Add:

```text
net.core.somaxconn = 4096
```

Take Effect:

```bash
sysctl -p
```

---

## Scenario 23: Viewing tcp_max_syn_backlog

```bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
```

Explanation:

```text
tcp_max_syn_backlog
→ Upper limit for semi-connection queues
```

Temporary Adjustment:

```bash
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
```

Permanent Configuration:

```bash
vi /etc/sysctl.conf
```

Add:

```text
net.ipv4.tcp_max_syn_backlog = 4096
```

Take Effect:

```bash
sysctl -p
```

---

## Scenario 24: Viewing ListenDrops and ListenOverflows

### Command

```bash
netstat -s | grep -i listen
```

Or:

```bash
nstat | grep -i Listen
```

You can also view:

```bash
cat /proc/net/netstat | grep -E "Listen|TcpExt"
```

Focus on:

```text
ListenOverflows
ListenDrops
```

If it continues to grow, it may indicate:

```text
Service listening line overflowing.

Apply accept Untimely

backlog Not enough.

Inadequate application processing capacity
```

---

## IX. Connection Count and File Descriptors

---

## Scenario 25: What Does Excessive Connections Affect

A large number of connections may lead to:

```text
File description deplete

Increase in memory consumption

Connect Queue Pressure

Thread or co-ordinated pressure

Slower application response

New connection creation failed

too many open files
```

---

## Scenario 26: Viewing System Socket Statistics

```bash
ss -s
```

Viewing TCP Connections:

```bash
ss -ant | wc -l
```

Statistical by Status:

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

---

## Scenario 27: Viewing Open File Count per Process

```bash
ls /proc/PID/fd | wc -l
```

Viewing Limits:

```bash
cat /proc/PID/limits | grep "Max open files"
```

Viewing System Limits:

```bash
ulimit -n
```

Viewing System-Level File Handles:

```bash
cat /proc/sys/fs/file-nr
```

Viewing Maximum File Handles:

```bash
cat /proc/sys/fs/file-max
```

---

## Scenario 28: too many open files

Common Log:

```text
Too many open files
```

Troubleshooting:

```bash
journalctl -u Service Name -n 100
```

```bash
grep -i "too many open files" Apply Log
```

```bash
ls /proc/PID/fd | wc -l
```

```bash
cat /proc/PID/limits | grep "Max open files"
```

Handling Direction:

```text
Add service file description limit

Check the connection leak.

Check file handle leak

Optimizing Connect Pools

Fix application not closed file or socket
```

---

## X. Local Temporary Port Exhaustion

---

## Scenario 29: What is a Local Temporary Port

When the local machine acts as a client accessing a remote service, it will use a local temporary port.

For example:

```text
Here. 10.0.0.10:45678
→ Visits 10.0.0.20:3306
```

Where `45678` is the local temporary port.

---

## Scenario 30: Viewing Temporary Port Range

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

Common Output:

```text
32768 60999
```

Indicates the available temporary port range.

---

## Scenario 31: Temporary Port Exhaustion Phenomenon

Common Phenomenon:

```text
Cannot assign requested address

connect failed

Mass TIME_WAIT

Failed to access downstream as client

Too many short connections

A target. IP:Port Very many connections.
```

Troubleshooting:

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -ant | awk 'NR>1 {print $4}' | awk -F: '{print $NF}' | sort | uniq | wc -l
```

Statistical by Target:

```bash
ss -ant | awk 'NR>1 {print $5}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 32: Handling Direction for Temporary Port Exhaustion

Handling Direction:

```text
Use connect pool

Open keepalive

Reduce Short Connections

Reduce abnormal retry

Expand temporary port range

Increased number of client machines

Optimization of the call

Confirm. TIME_WAIT Is it unusual?
```

Temporary Adjustment of Port Range:

```bash
sysctl -w net.ipv4.ip_local_port_range="10000 65000"
```

Permanent Configuration:

```bash
vi /etc/sysctl.conf
```

Add:

```text
net.ipv4.ip_local_port_range = 10000 65000
```

Take Effect:

```bash
sysctl -p
```

Production Note:

```text
Extending the port range is only a means of mitigation
Root causes are usually too many short connections or too many connections to reuse
```

---

## XI. conntrack Troubleshooting

---

## Scenario 33: What is conntrack

conntrack is the Linux kernel connection tracking mechanism.

Commonly used for:

```text
iptables NAT

Docker Network

Kubernetes Service

Gateway Forward

Firewall status tracking
```

It records connection states.

If the conntrack table is full, it may lead to:

```text
New connection failed

Request random timeout

DNS Query failed

Container network abnormal.

Kubernetes Service Access anomalies

NAT Transfer abnormal.

Log appearance nf_conntrack: table full
```

---

## Scenario 34: Viewing Current conntrack Count and Maximum

Viewing Current Count:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
```

Viewing Maximum Value:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
```

Calculating Usage Rate:

```bash
echo "$(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
```

---

## Scenario 35: Viewing conntrack Table Full Logs

```bash
dmesg -T | grep -i conntrack
```

```bash
journalctl -k | grep -i conntrack
```

Common Logs:

```text
nf_conntrack: table full, dropping packet
```

Explanation:

```text
conntrack Fill up, kernel starts discarding new connection data Package
```

---

## Scenario 36: Using conntrack Command to View

Install Tool:

Ubuntu/Debian:

```bash
apt install -y conntrack
```

RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y conntrack-tools
```

Or:

```bash
dnf install -y conntrack-tools
```

Viewing Statistics:

```bash
conntrack -S
```

Viewing Connections:

```bash
conntrack -L | head
```

Statistical by Protocol:

```bash
conntrack -L -p tcp | wc -l
```

---

## Scenario 37: Adjusting conntrack Maximum

Temporary Adjustment:

```bash
sysctl -w net.netfilter.nf_conntrack_max=262144
```

Permanent Configuration:

```bash
vi /etc/sysctl.conf
```

Add:

```text
net.netfilter.nf_conntrack_max = 262144
```

Take Effect:

```bash
sysctl -p
```

Production Note:

```text
Bigger conntrack Increased memory consumption.
We can't just make it bigger than Zagan.
```

---

## Scenario 38: Common Causes of conntrack Full

Common Causes:

```text
Too many short connections

Mass NAT Forward

Large traffic in container network

Kubernetes Service High number of requests

Unusual retesting storm.

DNS The query is abnormal.

External scanning or attack

TIME_WAIT / UDP Too many connection records

conntrack It doesn't make sense.
```

Handling Direction:

```text
Confirm connection source

Optimizing Short Connections

Lower Retry

Connection reuse

Embracing gateway nodes

Adjustment conntrack Upper limit

Optimizing overtime

Check out irregular traffic.
```

---

## XII. TCP Retransmission Troubleshooting

---

## Scenario 39: What Does TCP Retransmission Indicate

TCP retransmission typically indicates:

```text
The package sent was not confirmed

Or... ACK Lost

Leads the sender to retransmit the package
```

Common Causes:

```text
Network Throw

There's a chain.

Back-to-back processing slow

Dropped the net.

Firewall or security equipment interference

MTU Problem

The quality of the interplanetary link is poor

Cloud network vibrating
```

---

## Scenario 40: Viewing TCP Retransmission Statistics

Use `netstat -s`:

```bash
netstat -s | grep -i retrans
```

Use `nstat`:

```bash
nstat | grep -i retrans
```

Use `sar`:

```bash
sar -n TCP,ETCP 1 5
```

Focus on:

```text
retrans/s
```

---

## Scenario 41: tcpdump Capturing Retransmissions

Capture for specific host and port:

```bash
tcpdump -i any -nn host ObjectiveIP and port Port -w /tmp/tcp-retrans.pcap
```

After saving, you can view with Wireshark:

```text
TCP Retransmission

TCP Dup ACK

TCP Out-Of-Order

TCP Previous segment not captured
```

You can also directly observe:

```bash
tcpdump -i any -nn host ObjectiveIP and port Port
```

---

## Scenario 42: Retransmission Troubleshooting Direction

If retransmissions are obvious, continue to check:

```bash
ip -s link
```

```bash
sar -n DEV 1 5
```

```bash
mtr ObjectiveIP
```

```bash
ping -c 100 ObjectiveIP
```

```bash
tracepath ObjectiveIP
```

Possible Directions:

```text
IM card dropped / errors

The middle link is missing.

Overload

Quality of cross-barrel links

MTU Black Hole

Security equipment dropped.
```

---

## XIII. MTU Issue Troubleshooting

---

## Scenario 43: Common Phenomena of MTU Issues /think

MTU Common Issues:

```text
The bag can work. The big bag can't work.

ping Pumpkin, drop the bag.

curl Hold it!

TLS The handshake failed.

scp / rsync Big file transmission jammed.

Cross VPN / Tunnel access is unusual.

Intersectional communication across container network

Timeout when certain interfaces return big response
```

---

## Scenario 44: Check Network Interface MTU

```bash
ip link
```

Check specified network interface:

```bash
ip link show eth0
```

Common MTU:

```text
1500
1450
1400
9000
```

---

## Scenario 45: Use ping to Test MTU

Linux prohibits fragmentation testing:

```bash
ping -M do -s 1472 ObjectiveIP
```

Note:

```text
1472 + 28 Bytes IP/ICMP Head
≈ 1500 MTU
```

If failed, can gradually reduce:

```bash
ping -M do -s 1400 ObjectiveIP
```

```bash
ping -M do -s 1300 ObjectiveIP
```

---

## Scenario 46: tracepath Check Path MTU

```bash
tracepath ObjectiveIP
```

Purpose:

```text
Try to find the path MTU
```

Suitable for troubleshooting MTU issues in cross-segment, VPN, tunnel, and cloud network links.

---

## Scenario 47: MTU Handling Direction

Handling direction:

```text
Unified links MTU

Adjust the tunnel. MTU

Adjust container network MTU

Avoid black holes in paths

Check if the firewall is abandoned. ICMP Fragmentation Needed

Adjust the card MTU

Adjustment CNI MTU
```

Temporarily adjust network interface MTU:

```bash
ip link set dev eth0 mtu 1450
```

Production note:

```text
Modify MTU Could affect current connection
The production environment requires maintenance of windows and rollback programmes
```

---

## Fourteen, tcpdump Packet Capture Analysis

---

## Scenario 48: Capture on Specific Port

```bash
tcpdump -i any -nn port 8080
```

Capture on specific host and port:

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080
```

Capture fixed number:

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -c 100
```

Save:

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -w /tmp/test.pcap
```

---

## Scenario 49: Capture TCP SYN

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'
```

Specific port:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0 and port 8080'
```

Purpose:

```text
Observation TCP Connection creation request
```

---

## Scenario 50: Capture SYN but Exclude SYN-ACK

Only capture client initial SYN:

```bash
tcpdump -i any -nn 'tcp[tcpflags] == tcp-syn'
```

Purpose:

```text
Watch who's initiating the connection.
```

---

## Scenario 51: Capture FIN / RST

Capture FIN:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-fin != 0'
```

Capture RST:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0'
```

Specific port:

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0 and port 8080'
```

Purpose:

```text
Determines whether the connection has been proactively reset

To determine who initiated the shutdown.

The location's out of line.
```

---

## Scenario 52: Three-way Handshake Packet Capture Judgment

Normal three-way handshake:

```text
Client → Service:SYN

Service → Client:SYN, ACK

Client → Service:ACK
```

Capture packets:

```bash
tcpdump -i any -nn host ClientIP and port Service Port
```

Judgment:

```text
Only SYN
→ Request arrived or sent, but no response was seen

Yes. SYN and SYN-ACKNothing. ACK
→ Client failed to shake hands or return package abnormally

Yes. SYNI don't know.SYN-ACKI don't know.ACK
→ TCP Connection established

No data after handshake
→ Application layers may not send requests or be jammed
```

---

## Scenario 53: Four-way Close Packet Capture Judgment

Normal closing process is roughly:

```text
Send by side FIN

The other side. ACK

Send it to the other side. FIN

One side. ACK
```

If large amounts of `CLOSE_WAIT`:

```text
End FIN Already arrived

No application on this machine. close
```

If large amounts of `TIME_WAIT`:

```text
It's usually a voluntary shutdown.
```

Capture packets:

```bash
tcpdump -i any -nn host EndIP and port Port
```

---

## Fifteen, Network Interface dropped / errors Troubleshooting

---

## Scenario 54: Check Network Interface Statistics

```bash
ip -s link
```

Check specified network interface:

```bash
ip -s link show eth0
```

Focus on:

```text
RX errors

TX errors

RX dropped

TX dropped

overruns

carrier
```

---

## Scenario 55: Common Causes of dropped / errors

Common causes:

```text
Cybercard Line Full

The system can't handle it.

Driver issues

Virtual network card anomaly

Network congestion

Switch port problem

MTU Inconsistencies

Flows surged

There's too many buns.

The soft cut won't work.
```

Continue troubleshooting:

```bash
sar -n DEV 1 5
```

```bash
mpstat -P ALL 1 5
```

```bash
cat /proc/softirqs
```

```bash
ethtool -S eth0
```

---

## Scenario 56: Check Network Interface Driver and Speed

```bash
ethtool eth0
```

Check network interface statistics:

```bash
ethtool -S eth0
```

If no ethtool:

Ubuntu / Debian:

```bash
apt install -y ethtool
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y ethtool
```

Or:

```bash
dnf install -y ethtool
```

---

## Sixteen, Typical Advanced Network Troubleshooting Scenarios

---

## Scenario 57: Port Listening is Normal, but Client Connection Timeout

Troubleshooting path:

```text
Service ss Watch and listen.

→ Client nc Test

→ Service tcpdump Look. SYN Whether or not to arrive

→ Watch the firewall. / Security team

→ Look. SYN_RECV Whether to stack

→ Look. listen Is the queue full?

→ See if it's applied. accept Timely
```

Commands:

```bash
ss -lntp | grep Port
```

```bash
nc -zv -w 2 ServiceIP Port
```

```bash
tcpdump -i any -nn port Port
```

```bash
ss -ant state syn-recv
```

```bash
ss -lntp | grep Port
```

```bash
netstat -s | grep -i listen
```

---

## Scenario 58: Interface Occasional Timeout

Troubleshooting direction:

```text
Is there any? TCP Repeat

Whether to connect queue overflow

Whether or not conntrack Full

Whether or not DNS Occasional failure

Exhausted upstream or downstream connectors

Whether or not the card dropped Growth

Whether to apply thready pool full
```

Commands:

```bash
sar -n TCP,ETCP 1 5
```

```bash
nstat | grep -i retrans
```

```bash
ss -s
```

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
```

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
```

```bash
ip -s link
```

```bash
tcpdump -i any -nn host EndIP and port Port -w /tmp/timeout.pcap
```

---

## Scenario 59: Large Amount of CLOSE_WAIT

Troubleshooting:

```bash
ss -antp state close-wait
```

```bash
ss -antp state close-wait | wc -l
```

```bash
ss -antp state close-wait | grep users | head
```

Check processes:

```bash
ps -fp PID
```

```bash
ls /proc/PID/fd | wc -l
```

```bash
lsof -p PID | grep TCP | head
```

Handling direction:

```text
Apply connection leak

Code not closed

Connection pool anomaly

Thread's stuck.

I'm going to stop the bleeding temporarily.

Follow-up repair code or parameters
```

---

## Scenario 60: Large Amount of TIME_WAIT

Troubleshooting:

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | sort | uniq -c | sort -nr | head
```

Handling direction:

```text
Confirm if there are too many short connections

Confirm if we're actively closing.

Enable Connection Pool

Enable keepalive

Reduce retrying

Optimise call frequency

Check if the port is run out.
```

---

## Scenario 61: conntrack Table Full

Troubleshooting:

```bash
dmesg -T | grep -i conntrack
```

```bash
journalctl -k | grep -i conntrack
```

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
```

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
```

If using conntrack tool:

```bash
conntrack -S
```

Handling direction:

```text
A temporary increase. nf_conntrack_max

Check for unusual connections.

Reduce Short Connections

Optimization NAT / Container network traffic

Enhanced gateway or node

Reduce abnormal retry
```

---

## Scenario 62: Suspected MTU Issue

Phenomenon:

```text
The bag won't work.

curl Hold it!

TLS The handshake failed.

Big file transfer abnormal.

Cross VPN Or the tunnel is abnormal.
```

Troubleshooting:

```bash
ip link show eth0
```

```bash
ping -M do -s 1472 ObjectiveIP
```

```bash
ping -M do -s 1400 ObjectiveIP
```

```bash
tracepath ObjectiveIP
```

Handling direction:

```text
Align the links. MTU

Adjust the tunnel. MTU

Adjustment CNI MTU

Allow ICMP Fragmentation Needed

Unified links at both ends and in the middle MTU
```

---

## Seventeen, Container and Kubernetes Scenario Supplement

---

## Scenario 63: Docker Network Connection Abnormality

Troubleshooting:

```bash
docker ps
```

```bash
docker network ls
```

```bash
docker network inspect bridge
```

```bash
iptables -t nat -L -n -v
```

```bash
ss -ant
```

```bash
conntrack -S
```

Common directions:

```text
Docker bridge Net conflicts

iptables NAT The rules are abnormal.

Container port map anomaly

conntrack Full

Containers DNS Unusual

only listening in containers 127.0.0.1
```

---

## Scenario 64: Kubernetes Service Access Abnormality

Troubleshooting:

```bash
kubectl get svc -A
```

```bash
kubectl get endpoints -A
```

```bash
kubectl get pod -A -o wide
```

Check on nodes:

```bash
iptables -t nat -L -n -v
```

```bash
ipvsadm -L -n --stats
```

```bash
conntrack -S
```

```bash
dmesg -T | grep -i conntrack
```

Common directions:

```text
Service None endpoints

Pod Oh, no. Ready

kube-proxy The rules are abnormal.

iptables / ipvs Unusual

conntrack Full

CNI Network anomaly

NetworkPolicy Block
```

---

## Eighteen, Production Troubleshooting Notes

---

## 1. Do Not Use Only ping to Judge Network

`ping` only represents ICMP layer results.

Also combine:

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
curl -v http://Destination Address
```

```bash
ss -ant
```

```bash
tcpdump
```

---

## 2. Do Not Blindly Optimize TIME_WAIT

Large amounts of TIME_WAIT are not necessarily a fault.

First confirm:

```text
Whether to cause port depletion

Whether to cause connection failure

Compliance with the Business Short Connection Model

Is it possible to connect the pool and keepalive Optimization
```

---

## 3. CLOSE_WAIT is More Application-related

Large amounts of CLOSE_WAIT are usually not kernel parameter issues, but:

```text
Apply Unclosed Connections

Connection pool leak

Code problem.

Thread's stuck.
```

Prioritize checking processes and application logs.

---

## 4. conntrack Full Cannot Only Increase Parameters

Increasing parameters is just a temporary fix.

Also check:

```text
Connect Source

Connection Type

Too many short connections

Is there an abnormal retry?

Whether to attack or scan

Is there too much pressure on the joint?
```

---

## 5. tcpdump Should Control Scope

Production packet capture recommends adding:

```text
host

port

-c

-w
```

Example: /think

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -c 200 -w /tmp/debug.pcap
```

Avoid:

```text
I'll catch the bag for a long time.

pcap Write Disk Full

Overjacking during peak period affects performance
```

---

## 6. Backup and Evaluate Before Modifying Kernel Parameters

Check before modification:

```bash
sysctl -a | grep Parameter Name
```

Suggest recording before change:

```bash
sysctl -a > /tmp/sysctl-before-$(date +%F-%H%M%S).txt
```

Verify after modification:

```bash
sysctl -p
```

---

## 19. Summary of Common Commands in This Article

---

## TCP States

```bash
ss -s
```

```bash
ss -ant
```

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}'
```

```bash
ss -ant state established
```

```bash
ss -ant state time-wait
```

```bash
ss -ant state close-wait
```

```bash
ss -ant state syn-sent
```

```bash
ss -ant state syn-recv
```

---

## Listening and Queues

```bash
ss -lnt
```

```bash
ss -lntp
```

```bash
ss -lntp | grep 8080
```

```bash
cat /proc/sys/net/core/somaxconn
```

```bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
```

```bash
netstat -s | grep -i listen
```

```bash
nstat | grep -i Listen
```

---

## TIME_WAIT

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```

---

## CLOSE_WAIT

```bash
ss -antp state close-wait
```

```bash
ss -antp state close-wait | wc -l
```

```bash
ss -antp state close-wait | grep users | head
```

```bash
ls /proc/PID/fd | wc -l
```

```bash
lsof -p PID | grep TCP | head
```

---

## SYN States

```bash
ss -ant state syn-sent
```

```bash
ss -ant state syn-recv
```

```bash
ss -ant state syn-recv | wc -l
```

```bash
ss -ant state syn-recv | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head
```

---

## File Descriptors

```bash
ulimit -n
```

```bash
cat /proc/PID/limits | grep "Max open files"
```

```bash
ls /proc/PID/fd | wc -l
```

```bash
cat /proc/sys/fs/file-nr
```

```bash
cat /proc/sys/fs/file-max
```

---

## Temporary Ports

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

```bash
sysctl -w net.ipv4.ip_local_port_range="10000 65000"
```

---

## conntrack

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
```

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
```

```bash
dmesg -T | grep -i conntrack
```

```bash
journalctl -k | grep -i conntrack
```

```bash
conntrack -S
```

```bash
conntrack -L | head
```

```bash
conntrack -L -p tcp | wc -l
```

---

## TCP Retransmissions

```bash
netstat -s | grep -i retrans
```

```bash
nstat | grep -i retrans
```

```bash
sar -n TCP,ETCP 1 5
```

---

## MTU

```bash
ip link
```

```bash
ip link show eth0
```

```bash
ping -M do -s 1472 ObjectiveIP
```

```bash
ping -M do -s 1400 ObjectiveIP
```

```bash
tracepath ObjectiveIP
```

```bash
ip link set dev eth0 mtu 1450
```

---

## tcpdump

```bash
tcpdump -i any -nn port 8080
```

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080
```

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -c 100
```

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -w /tmp/test.pcap
```

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'
```

```bash
tcpdump -i any -nn 'tcp[tcpflags] == tcp-syn'
```

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-fin != 0'
```

```bash
tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0'
```

---

## Network Card Statistics

```bash
ip -s link
```

```bash
ip -s link show eth0
```

```bash
sar -n DEV 1 5
```

```bash
ethtool eth0
```

```bash
ethtool -S eth0
```

---

## Docker / Kubernetes

```bash
docker network ls
```

```bash
docker network inspect bridge
```

```bash
iptables -t nat -L -n -v
```

```bash
kubectl get svc -A
```

```bash
kubectl get endpoints -A
```

```bash
kubectl get pod -A -o wide
```

```bash
ipvsadm -L -n --stats
```

---

## Tool Installation

Ubuntu / Debian install sysstat:

```bash
apt install -y sysstat
```

RHEL / CentOS / Rocky / AlmaLinux install sysstat:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Ubuntu / Debian install conntrack:

```bash
apt install -y conntrack
```

RHEL / CentOS / Rocky / AlmaLinux install conntrack-tools:

```bash
yum install -y conntrack-tools
```

Or:

```bash
dnf install -y conntrack-tools
```

Ubuntu / Debian install ethtool:

```bash
apt install -y ethtool
```

RHEL / CentOS / Rocky / AlmaLinux install ethtool:

```bash
yum install -y ethtool
```

Or:

```bash
dnf install -y ethtool
```

---

## 20. One-Sentence Summary

The core of advanced network troubleshooting isn't just checking if ports are open, but determining:

```text
Where's the connection card? TCP Status

The team's not full.

Has the connection leaked?

Has the port run out?

conntrack Are you full?

Is the network re-routed and thrown?

Have the request bag arrived?

Response package returned
```

TCP state determination:

```text
SYN_SENT
→ We're launching the connection, but it's not finished for the end. Response

SYN_RECV
→ Roger that. SYNWaiting for the client to shake hands

ESTABLISHED
→ Connection established

TIME_WAIT
→ The active shut-down party awaits the complete release of the connection.

CLOSE_WAIT
→ End is closed, this application is not closed socket
```

Connection queue troubleshooting:

```text
ss -lnt Look. Recv-Q / Send-Q

netstat -s Look. ListenDrops / ListenOverflows

somaxconn Look at all the lines.

tcp_max_syn_backlog Look at the limit of the semi-connection queue.
```

conntrack troubleshooting:

```text
nf_conntrack_count Look at the current connection trail

nf_conntrack_max Look at the ceiling.

dmesg / journalctl Look. table full

conntrack -S Look at the statistics.
```

Packet capture determination:

```text
Only SYN
→ There's no response to the end or the middle bag.

SYN + SYN-ACK + ACK
→ Three handshakes.

RST
→ Connection Resetd

FIN
→ Normal closing process

Repeat
→ Could be throwing away bags, stuffing,MTU or to the end slow
```

Production recommendations:

```text
Don't just use it. ping Watch the network
Don't. TIME_WAIT Directly as a malfunction.
Mass CLOSE_WAIT Prioritize application of connection leaks
conntrack We can't just turn the parameters.
We'll add the bag. host / port / -c Scope of limitations
Backup and evaluation before changing kernel parameters
Network questions are viewed simultaneously with client, service, intermediate links and application logs
```