# 11-Advanced Network Troubleshooting: TCP States, Connection Queues, conntrack, TIME_WAIT, and Packet Capturing Analysis

# Linux # Operations and Maintenance # SRE # Network Troubleshooting # TCP # Connection Queues # conntrack # TIME_WAIT # CLOSE_WAIT # SYN_RECV # tcpdump # MTU # Performance Analysis

---

## Recommended Reading Path

01-Linux Basics and Host Operations and Maintenance/01-Host Troubleshooting/11-Advanced Network Troubleshooting: TCP States, Connection Queues, conntrack, TIME_WAIT, and Packet Capturing Analysis.md

---

## I. Document Overview

This document outlines advanced methods for network troubleshooting on Linux hosts. It focuses not only on determining whether ports are accessible but also on analyzing various aspects:

- TCP connection states
- `SYN_SENT`
- `SYN_RECV`
- `ESTABLISHED`
- `TIME_WAIT`
- `CLOSE_WAIT`
- Half-open connection queues
- Full connection queues
- `listen backlog`
- `somaxconn`
- `tcp_max_syn_backlog`
- The conntrack table
- NF_conntrack table fullness
- Exhaustion of local temporary ports
- TCP retransmissions
- Packet capturing for TCP three-way handshakes and four-way closures
- MTU issues
- Network packet loss identification
- In-depth troubleshooting when service ports are listening but services are not functioning

This document is part of the advanced network performance analysis series within the host troubleshooting category.

In previous article 06, we covered network connectivity, port checking, routing, and traffic analysis. This article delves deeper into TCP, the kernel's network stack, and packet capturing techniques.

The objectives are:

- To understand common TCP states
- To determine at which stage a connection is stuck
- To troubleshoot full connection queues
- To identify abnormal values for TIME_WAIT and CLOSE_WAIT
- To diagnose network issues caused by a full conntrack table
- To preliminarily assess exhaustion of local ports
- To use tcpdump to verify three-way handshakes, retransmissions, packet loss, and MTU problems
- To develop an advanced understanding of network troubleshooting approaches suitable for SRE interviews

---

## II. General Approach to Advanced Network Troubleshooting

Regular network troubleshooting focuses on:

```text
Whether the IP is accessible
Whether ports are reachable
Whether services are listening
Whether routing is correct
Whether firewalls allow traffic
```

Advanced network troubleshooting also involves examining:

```text
At which TCP state the connection is stuck
Whether connection queues are full
Whether there are a large number of TIME_WAIT or CLOSE_WAIT connections
Whether there are many SYN_RECV connections
Whether TCP retransmissions occur
Whether the conntrack table is full
Whether local temporary ports are being exhausted
Whether MTU issues exist
Whether there are network interface errors or packet drops
Whether request packets are reaching their destination
Whether response packets are returning
Whether three-way handshakes are complete
Whether four-way closures are normal
```

Recommended troubleshooting steps include:

```text
ss -s
→ View the overall status of TCP connections
ss -ant
→ Analyze the distribution of specific TCP states
ss -lnt
→ Check listening ports and connection queues
netstat -s / nstat
→ Identify TCP/IP protocol stack errors and retransmissions
conntrack / nf_conntrack
→ Examine the connection tracking tables
ip -s link
→ Check for network interface error packets and packet losses
sar -n TCP,ETCP,DEV,SOCK
→ View statistics on TCP, errors, network interfaces, and sockets
tcpdump
→ Capture packets to verify requests, responses, retransmissions, handshakes, and MTU issues
```

---

## III. Basics of TCP States

---

## Scenario 1: Viewing TCP Connection Status Statistics

### Command

```bash
ss -s
```

### Purpose

This command displays overall socket statistics.

Focus on the following fields:

```text
TCP
ESTAB
TIME-WAIT
CLOSE-WAIT
SYN-SENT
SYN-RECV
```

It is useful for quickly assessing:

```text
Whether the current number of connections is abnormal
Whether there are a large number of TIME_WAIT connections
Whether CLOSE_WAIT connections are accumulating
Whether SYN_RECV connections are unusual
Whether many connections are in abnormal states
```

---

## Scenario 2: Viewing All TCP Connections

### Command

```bash
ss -ant
```

### Parameter Explanation

```text
-a
→ Show all connections
-n
→ Display numbers instead of resolving domain names and port names
-t
→ Limit the output to TCP connections
```

---

## Scenario 3: Counting TCP Connections by State

### Command

```bash
ss -ant | awk 'NR>1 {count[$1]++} END {for (state in count) print state, count[state]}```bash
ss -ant state time-wait | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```## X. Exhaustion of Local Temporary Ports

---

## Scenario 29: What are Local Temporary Ports?

When the local machine acts as a client to access remote services, local temporary ports are used.

For example:

```text
Local machine 10.0.0.10:45678
→ Accessing 10.0.0.20:3306
```

In this case, `45678` is the local temporary port.

---

## Scenario 30: Viewing the Range of Temporary Ports

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

Common output:

```text
32768 60999
```

This indicates the available range of temporary ports.

---

## Scenario 31: Symptoms of Temporary Port Exhaustion

Common symptoms include:

```text
Cannot assign requested address
Connect failed
Numerous TIME_WAIT entries
Failure to access downstream targets as a client
Excessive short-lived connections
Abnormally high number of connections to a specific target IP:Port combination
```

Troubleshooting steps:

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -ant | awk 'NR>1 {print $4}' | awk -F: '{print $NF}' | sort | uniq | wc -l
```

Statistical analysis by target:

```bash
ss -ant | awk 'NR>1 {print $5}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 32: Solutions for Temporary Port Exhaustion

Possible solutions include:

```text
Using a connection pool
Enabling keepalive
Reducing the number of short-lived connections
Decreasing excessive retries
Increasing the range of temporary ports
Adding more client machines
Optimizing the way calls are made
Verifying if there are any abnormal TIME_WAIT entries
```

Temporary adjustment of the port range:

```bash
sysctl -w net.ipv4.ip_local_port_range="10000 65000"
```

Permanent configuration:

```bash
vi /etc/sysctl.conf
```

Add the following line:

```text
net.ipv4.ip_local_port_range = 10000 65000
```

To apply the changes:

```bash
sysctl -p
```

Production considerations:

```text
Increasing the port range is only a temporary fix. The root cause is usually excessive short-lived connections or insufficient connection reuse.
```

---

## XI. Troubleshooting with conntrack

---

## Scenario 33: What is conntrack?

conntrack is the Linux kernel's mechanism for tracking connections.

It is commonly used in:

```text
iptables NAT
Docker networks
Kubernetes Services
Gateway forwarding
Firewall status tracking
```

It keeps track of connection states. If the conntrack table becomes full, it may cause issues such as:

```text
Failure of new connections
Random request timeouts
DNS query failures
Abnormalities in container networks
Issues accessing Kubernetes Services
NAT forwarding errors
Logs displaying "nf_conntrack: table full"
```

---

## Scenario 34: Viewing the Current Number and Maximum Value of conntrack

To view the current number:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
```

To view the maximum value:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_max
```

To calculate the usage rate:

```bash
echo "$(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
```

---

## Scenario 35: Viewing Logs of conntrack Table Full

```bash
dmesg -T | grep -i conntrack
```

```bash
journalctl -k | grep -i conntrack
```

Common logs include:

```text
nf_conntrack: table full, dropping packet
```

This indicates that the kernel has started discarding new connection-related packets due to the table being full.

---

## Scenario 36: Using conntrack Commands for Analysis

To install the necessary tools:

For Ubuntu / Debian:

```bash
apt install -y conntrack
```

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y conntrack-tools
```

Or:

```bash
dnf install -y conntrack-tools
```

To view statistics:

```bash
conntrack -S
```

To list connections:

```bash
conntrack -L | head
```

To count connections by protocol:

```bash
conntrack -L -p tcp | wc -l
```

---

## Scenario 37: Adjusting the Maximum Value of conntrack

Temporary adjustment:

```bash
sysctl -w net.netfilter### Scenario 53: Packet Capture and Judgment for the Four-Way Handshake

The normal four-way handshake process is as follows:

- Client → Server: SYN
- Server → Client: SYN, ACK
- Client → Server: ACK

To capture packets during this process:

```bash
tcpdump -i any -nn host client_ip address and port server_port number
```

Judgment based on captured packets:

- Only SYN: The request has been sent or received, but no response has been seen.
- SYN and SYN-ACK but no ACK: The client either failed to complete the handshake or there is an issue with the packet return path.
- SYN, SYN-ACK, and ACK: The TCP connection has been established.
- No data after the handshake: It may be that the application layer has not sent a request or there is some blocking issue.

### Scenario 54: Checking Network Card Statistics

To view general network card statistics:

```bash
ip -s link
```

To check specific network cards, for example, `eth0`:

```bash
ip -s link show eth0
```

Pay attention to the following fields:

- RX errors
- TX errors
- RX dropped
- TX dropped
- Overruns
- Carrier

### Scenario 55: Common Causes of Packet Drops and Errors

Common reasons for packet drops and errors include:

- The network card queue is full.
- The system cannot process packets quickly enough.
- There are issues with the network card driver.
- Abnormalities with virtualized network cards.
- Network congestion.
- Problems with switch ports.
- Inconsistent MTU values.
- Sudden increases in traffic.
- Excessive number of small packets.
- The system cannot handle soft interrupts efficiently enough.

To further investigate these issues, you can use the following commands:

```bash
sar -n DEV 1 5
mpstat -P ALL 1 5
cat /proc/softirqs
ethtool -S eth0
```

### Scenario 56: Checking Network Card Drivers and Speeds

To view information about a network card driver:

```bash
ethtool eth0
```

To check the network card's statistics in more detail:

```bash
ethtool -S eth0
```

If `ethtool` is not available on your system, you can install it using the following commands for Ubuntu/Debian:

```bash
apt install -y ethtool
```

For RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y ethtool
```

Or for Fedora/RHEL:

```bash
dnf install -y ethtool
```Container port mapping exception

conntrack table is full

Container DNS exception

The service inside the container only listens on 127.0.0.1```bash
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

To install sysstat on Ubuntu / Debian:

```bash
apt install -y sysstat
```

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

To install conntrack on Ubuntu / Debian:

```bash
apt install -y conntrack
```

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y conntrack-tools
```

Or:

```bash
dnf install -y conntrack-tools
```

To install ethtool on Ubuntu / Debian:

```bash
apt install -y ethtool
```

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y ethtool
```

Or:

```bash
dnf install -y ethtool
```

---

## Twenty, One-Sentence Summary

The core of advanced network troubleshooting is not just to check if ports are open, but to determine:

- **In which TCP state the connection is stuck**.
- **Whether the queue is full**.
- **If there are any connection leaks**.
- **If ports are being exhausted**.
- **Whether conntrack is at capacity**.
- **If there are retransmissions or packet losses in the network**.
- **Whether request packets have arrived and response packets have been returned**.

**TCP State Identification:**

- **SYN_SENT**: The local system initiated a connection, but the other party has not responded.
- **SYN_RECV**: The local system received a SYN packet and is waiting for the client to complete the handshake.
- **ESTABLISHED**: The connection has been established.
- **TIME_WAIT**: The initiating party is waiting for the connection to be completely released.
- **CLOSE_WAIT**: The other party has closed the connection, but the local application has not closed the socket.

**Connection Queue Inspection:**

- Use `ss -lnt` to check `Recv-Q` and `Send-Q`.
- Use `netstat -s` to check `ListenDrops` and `ListenOverflows`.
- Check `somaxconn` for the maximum number of active connections.
- Verify `tcp_max_syn_backlog` for related half-open connection limits.

**Conntrack Inspection:**

- Use `nf_conntrack_count` to check the current number of connection tracks.
- Check `nf_conntrack_max` for the upper limit.
- Review `dmesg` or `journalctl` for messages about table fullness.
- Use `conntrack -S` to view statistics.

**Packet Capture Analysis:**

- A packet with only a SYN indicates that the other party may not be responding or there might be intermediate packet losses.
- A sequence of `SYN`, `SYN-ACK`, and `ACK` indicates a successful three-way handshake.
- A `RST` signal indicates that the connection was reset.
- A `FIN` signal signifies a normal closure process.
- Retransmissions may indicate packet losses, congestion, issues with the MTU, or slow processing on the other end.

**Production Recommendations:**

- **Do not rely solely on ping to assess network status.**
- **Do not immediately consider `TIME_WAIT` as a fault.**
- **If there are many `CLOSE_WAIT` entries, suspect potential application connection leaks.**
- **Simply increasing conntrack parameters may not solve the problem.**
- **When capturing packets, use filters like `host`, `port`, and `-c` to narrow down the scope of analysis.**
- **Before modifying kernel parameters, back them up and assess the possible impacts.**
- **For network issues, consider examining both client, server, intermediate links, and application logs simultaneously.**