# 06-Network Connectivity, Ports, Routing, and Traffic Troubleshooting

# Linux # Operations and Maintenance # Troubleshooting # Network Troubleshooting # Port Troubleshooting # Routing Troubleshooting # Traffic Troubleshooting # ss # nc # curl # tcpdump # traceroute # iftop

---

## Recommended Path

01-Linux Basics and Host Operations and Maintenance/01-Host Troubleshooting/06-Network Connectivity, Ports, Routing, and Traffic Troubleshooting.md

---

## I. Document Description

This document compiles commands related to **network connectivity, ports, routing, and traffic troubleshooting** on Linux hosts.

Key points include:

- Viewing network cards and IP addresses
- Basic connectivity tests
- ICMP connectivity troubleshooting
- TCP port connectivity testing
- HTTP/HTTPS service testing
- Listening port inspection
- TCP/UDP connection monitoring
- Route table examination
- Checking the actual routing to a target IP
- Network link path verification
- Network card traffic analysis
- Inspection of network error packets/dropped packets
- Continuous packet sending tests
- Continuous port testing
- tcpdump packet capture and validation
- End-to-end network connectivity verification

The goal is:

To determine whether the host's network is functioning properly

→ To verify if a target IP address is reachable

→ To confirm if a target port is open

→ To check if HTTP/HTTPS services are working correctly

→ To ascertain if local services are listening on specific ports

→ To identify the routing used to access the target

→ To view network card traffic and error packets

→ To use tcpdump to verify whether requests reach the destination and responses are returned

---

## II. General Approach to Network Troubleshooting

Don't rely solely on `ping` for diagnosing network issues.

Recommended troubleshooting sequence:

```text
Confirm local IP/network card status

→ Use ping to test basic connectivity

→ Use nc to test TCP ports

→ Use curl to test HTTP/HTTPS services

→ Use ss to check local listening ports

→ Use ip route to view the routing table

→ Use traceroute/tracepath to examine the path

→ Use sar/iftop/ip -s link to monitor traffic and packet loss

→ Use tcpdump to capture and verify requests and responses

→ Further investigate using firewalls, security groups, ACLs, and logs
```

Common troubleshooting steps:

```text
If ping fails:
→ Possible issues include network problems, routing issues, firewall restrictions, ICMP blocking, or security group settings.

If ping succeeds but ports are unreachable:
→ Possible reasons include services not being listed for listening, ports not open, firewalls, security groups, or incorrect listening addresses.

If ports are open but services fail:
→ Possible issues involve application protocols, HTTP status codes, certificate problems, backend service failures, or business logic errors.

If local listening is normal but external access fails:
→ Possible causes include firewalls, security groups, routing issues, listening addresses, or NAT settings.

If traffic is abnormal:
→ Check sar, iftop, ip -s link, and tcpdump for details.
```

---

## III. Common Network Issues

In production environments, network problems often manifest as follows:

```text
Unable to log in via SSH

Inaccessible service ports

Applications fail when connecting to databases

Third-party API calls time out

Nginx returns 502/504 errors

curl requests get stuck

DNS resolution is successful but connections fail

Ping succeeds but ports are unreachable

Local access is possible, but external access fails

External IP addresses can be reached, but domain names don't work

Services listen on 127.0.0.1, but external access fails

Sudden increase in network traffic

Network cards show dropped/packet errors

Abnormal cross-network segment access

Incomplete cloud security group rules
```

Common causes include:

```text
Incorrect IP configuration

Network cards not enabled

Routing errors

Wrong default gateway settings

Services not listening for connections

Services only listening on 127.0.0.1

Ports blocked by firewalls

Cloud security groups not allowing access

ACL/network policy restrictions

DNS resolution issues

Network link packet loss

Abnormalities with network devices or cloud networks

Excessive local connections

Bandwidth exhaustion
```

---

## IV. ip a: Viewing Network Cards and IP Addresses

---

## Scenario 1: Viewing All Network Cards and IP Addresses

### Command

```bash
ip a
```

or:

```bash
ip addr
```

### Purpose

This command displays the following information on the host:

```text
Network card names

Network card status

IP addresses

MAC addresses

IPv4/IPv6 addresses
```

Key points to check:

```text
Whether network cards are UP

Whether IP addresses are correct

The presence of multiple network cards or IPs

The existence of virtual network cards such as docker0/cni0/flannel/cali
``### Interface Detection

### Script Testing

### Avoid Long-Time BlockagesNetwork Device ACL

Cloud Network Policy

Packet Capture and Joint Debugging at Both Ends

Production Notes:

```text
Do not send packets frequently and for extended periods in a production environment to avoid affecting the link and target host.
```

---

## Scenario 38: Continuously Testing a Port

### Command

```bash
while true; do nc -zv 10.0.0.5 3306; sleep 1; done
```

Setting a timeout:

```bash
while true; do nc -zv -w 2 10.0.0.5 3306; sleep 1; done
```

### Purpose

Continuously check whether a certain port is reachable.

Suitable for:

```text
Observing when a port recovers during service restarts

Verifying after adjusting firewall rules

Checking after allowing access through security groups

Confirming whether requests reach the other end by capturing packets
```

---

## Scenario 39: Continuously Sending HTTP Requests

### Command

```bash
while true; do curl -I http://10.0.0.5:8080; sleep 1; done
```

With a timeout:

```bash
while true; do curl -m 3 -I http://10.0.0.5:8080; sleep 1; done
```

### Purpose

Continuously initiate HTTP requests.

Suitable for use in conjunction with:

```text
Server access logs

Server error logs

tcpdump packet capture

Nginx upstream configuration adjustments

Verification after service restarts
```

---

## XIV. tcpdump: Packet Capture to Verify Requests and Responses

---

## Scenario 40: Capturing Traffic from a Specific Network Card

### Command

```bash
tcpdump -i eth0
```

### Purpose

Capture network packets on a specified network card.

---

## Scenario 41: Not Parsing Hostnames and Port Names

### Command

```bash
tcpdump -i eth0 -nn
```

### Parameter Explanation

```text
-nn
→ Do not parse hostnames and port names; display only IP addresses and ports directly.
```

It is more recommended to add `-nn` during production troubleshooting to avoid speed impacts caused by DNS lookups.

---

## Scenario 42: Capturing Traffic from a Specific Host

### Command

```bash
tcpdump -i eth0 -nn host 10.0.0.5
```

### Purpose

Only capture traffic related to a specific IP address.

---

## Scenario 43: Capturing Traffic from a Specific Port

### Command

```bash
tcpdump -i eth0 -nn port 3306
```

To capture HTTP traffic:

```bash
tcpdump -i eth0 -nn port 80
```

To capture HTTPS traffic:

```bash
tcpdump -i eth0 -nn port 443
```

---

## Scenario 44: Capturing Traffic from a Specific Host and Port

### Command

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306
```

### Purpose

Suitable for accurately verifying:

```text
Whether traffic from a certain host to a specific port reaches the destination

Whether there is a response to the request

Whether a connection is established
```

---

## Scenario 45: Capturing Traffic from All Network Cards

### Command

```bash
tcpdump -i any -nn
```

To capture traffic from a specific port:

```bash
tcpdump -i any -nn port 8080
```

To capture traffic from a specific host and port:

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080
```

### Purpose

When it is uncertain which network card the traffic will use, `-i any` makes it more convenient.

---

## Scenario 46: Exiting After Capturing a Fixed Number of Packets

### Command

```bash
tcpdump -i eth0 -nn -c 20
```

To specify conditions:

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306 -c 20
```

### Parameter Explanation

```text
-c 20
→ Automatically exit after capturing 20 packets.
```

---

## Scenario 47: Saving Packets as a pcap File

### Command

```bash
tcpdump -i eth0 -nn -w /tmp/test.pcap
```

To specify a specific host and port:

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306 -w /tmp/mysql-testHTTP Status Code Exceptions

Authentication Failed

Certificate Issues

Backend Timeout

Protocol Mismatch

Application Logic Errors

Therefore, it is also necessary to check:

```bash
curl -v
```

```bash
journalctl -u service_name
```

```bash
tail -f application logs
```

---

## 3. Control the Scope When Capturing Packets

In a production environment, it is not recommended to use `tcpdump -i any` for an extended period:

It is better to add conditions:

```bash
tcpdump -i any -nn host target_IP and port port -c 100
```

Or save the packets to a file:

```bash
tcpdump -i any -nn host target_IP and port port -w /tmp/test.pcap
```

---

## 4. When Investigating High Traffic, Avoid Damaging Business Operations

When using `iftop`, `tcpdump`, or `sar` for troubleshooting, pay attention to the following:

```text
Do not capture all packets for an extended period.

Do not save pcap files on disks with insufficient space.

Do not perform high-traffic stress tests during peak hours.

Do not scan large portions of the production network segment.
```

---

## 5. For Hosts with Multiple Network Cards, Focus on Source IP and Outbound Interface

When troubleshooting machines with multiple network cards, it is essential to check:

```bash
ip route get target_IP
```

Confirm:

```text
Which is the actual source IP?

Which network card is used for the outbound connection?

Is the data passing through the expected gateway?
```

---

## Section 17: Summary of Commonly Used Commands

---

## Network Cards and IP Addresses

```bash
ip a
```

```bash
ip addr
```

```bash
ip addr show eth0
```

```bash
ip link
```

```bash
ip link show eth0
```

```bash
ip link set eth0 up
```

```bash
ip link set eth0 down
```

---

## Basic Connectivity Tests

```bash
ping target_IP
```

```bash
ping -c 4 target_IP
```

```bash
ping -s 1400 -i 0.2 target_IP
```

---

## Port Testing

```bash
nc -zv target_IP port
```

```bash
nc -zv -w 2 target_IP port
```

```bash
while true; do nc -zv -w 2 10.0.0.5 3306; sleep 1; done
```

---

## HTTP / HTTPS Testing

```bash
curl http://target_address
```

```bash
curl -I http://target_address
```

```bash
curl -v http://target_address
```

```bash
curl -vk https://target_address
```

```bash
curl -m 5 http://target_address
```

```bash
curl -L -I http://target_address
```

```bash
while true; do curl -m 3 -I http://10.0.0.5:8080; sleep 1; done
```

---

## Listening and Connection Tests

```bash
ss -tunlp
```

```bash
ss -antp
```

```bash
ss -s
```

```bash
ss -tunlp | grep 8080
```

```bash
netstat -tunlp
```

```bash
netstat -anp
```

---

## Routing

```bash
ip route
```

```bash
ip route get target_IP
```

```bash
traceroute target_IP
```

```bash
traceroute -n target_IP
```

```bash
tracepath target_IP
```

---

## Network Traffic Monitoring

```bash
sar -n DEV 1 5
```

```bash
iftop
```

```bash
iftop -n
```

```bash
iftop -i eth0
```

```bash
ip -s link
```

```bash
ip -s link show eth0
```

---

## Packet Capture

```bash
tcpdump -i eth0
```

```bash
tcpdump -i eth0 -nn
```

```bash
tcpdump -i eth0 -nn host 10.0.0.5
```

```bash
tcpdump -i eth0 -nn port 3306
```

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306
```

```bash
tcpdump -i any -nn
```

```bash
tcpdump -i any -nn port 8080
```

```bash
tcpdump