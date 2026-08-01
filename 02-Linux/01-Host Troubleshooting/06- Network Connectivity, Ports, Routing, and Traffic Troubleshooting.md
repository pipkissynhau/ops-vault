# 06-Network Connectivity, Ports, Routing, and Traffic Troubleshooting

#Linux #Transport #TheBarrier. #NetworkChecking #PortCheck #RouteControl. #FlowChecking #ss #nc #curl #tcpdump #traceroute #iftop

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/06-Network Connectivity, Ports, Routing, and Traffic Troubleshooting.md

---

## I. Document Description

This document organizes commands related to **network connectivity, port, routing, and traffic troubleshooting** on Linux hosts.

Key focuses include:

- View network interface and IP
- Basic connectivity testing
- ICMP connectivity troubleshooting
- TCP port connectivity troubleshooting
- HTTP/HTTPS service testing
- Port listening inspection
- TCP/UDP connection inspection
- Routing table inspection
- Actual routing path to target IP
- Network link path troubleshooting
- Network interface traffic inspection
- Network error packets/dropped packets inspection
- Continuous packet sending testing
- Continuous port testing
- tcpdump packet capture verification
- End-to-end network connectivity debugging

Goals:

- Determine if the host network is normal
- Determine if the target IP is reachable
- Determine if the target port is open
- Determine if HTTP/HTTPS services are normal
- Determine if local services are listening on ports
- Determine the routing path to the target
- View network interface traffic and error packets
- Verify via tcpdump if requests arrive and responses return

---

## II. Network Troubleshooting General Approach

Do not rely solely on `ping` for network issues.

Recommended troubleshooting order:

```text
Confirm this machine. IP / Card Status

→ ping Measuring basic connectivity

→ nc Measure TCP Port

→ curl Measure HTTP / HTTPS Services

→ ss View this machine listening

→ ip route View route

→ traceroute / tracepath View Path

→ sar / iftop / ip -s link View traffic and packages

→ tcpdump Capture package validation request and response

→ Combining firewalls, security units,ACLLogs continue to locate
```

CommonDiversion:

```text
ping It's not working.
→ Could be the network, route, firewall,ICMP Banned, security team.

ping It doesn't make sense.
→ Could be the service was unheard, the port was not open, the firewall, the security team, the wired address was wrong.

Portable but business abnormal.
→ Maybe it's an application protocol.HTTP Status codes, certificates, back-end services, business logic issues

We're listening to normal, but no outside access.
→ Could be firewalls, security units, routers, listening addresses,NAT Problem

Flow abnormal.
→ Look. sarI don't know.iftopI don't know.ip -s linkI don't know.tcpdump
```

---

## III. Common Network Issues

In production environments, network issues often manifest as:

```text
SSH I can't log in.

Service port access is not available

Apply connection database timeout

Call third-party interface timed out

Nginx Back 502 / 504

curl Please hold.

DNS Can parse but connect failed

ping It doesn't make sense.

I can't get an external visit.

External access IPbut domain name access is abnormal

Service wired. 127.0.0.1External visits failed

Network traffic has suddenly increased.

The card. dropped / errors

Cross-net access anomaly

Clearing the clouds is incomplete.
```

Common causes include:

```text
IP Configure Error

Netcard not enabled

Path error

Default Gateway Error

Service not monitored

Services only listen. 127.0.0.1

Port blocked by firewall.

Cloud Security Unit not released

ACL / Network strategy blocked

DNS Parsing anomaly

Network links dropped.

Network equipment or cloud network anomalies

Too many connections.

The bandwidth is full.
```

---

## IV. ip a: View Network Interface and IP

---

## Scenario 1: View All Network Interfaces and IPs

### Command

```bash
ip a
```

Or:

```bash
ip addr
```

### Purpose

View the host's:

```text
Netcard Name

Card Status

IP Address

MAC Address

IPv4 / IPv6 Address
```

Focus on:

```text
Do you want a card? UP

IP Address correct

Do you have multiple cards?

Are there more than one? IP

Is there any? docker0 / cni0 / flannel / cali Wait for the virtual card.
```

---

## Scenario 2: View Specific Network Interface

### Command

```bash
ip addr show eth0
```

Or:

```bash
ip a show eth0
```

Example:

```bash
ip addr show ens33
```

### Purpose

View information for a specific network interface.

Suitable for confirming:

```text
Business card IP Correct?

Manage whether or not the card UP

Is there an unusual address for the card?

Visibility of Virtual Machine Card
```

---

## Scenario 3: View Network Interface Link Status

### Command

```bash
ip link
```

View specific network interface:

```bash
ip link show eth0
```

### Common Status

```text
UP
→ Webcard Enabled

DOWN
→ Netcard not enabled

LOWER_UP
→ The chain layer is fully connected.
```

Enable network interface:

```bash
ip link set eth0 up
```

Disable network interface:

```bash
ip link set eth0 down
```

Production Note:

```text
Don't make yourself comfortable. down Production net cards
Could be. SSH Disconnection or business interruption
```

---

## V. ping: Basic Connectivity Testing

---

## Scenario 4: Test if Target IP is Reachable

### Command

```bash
ping ObjectiveIP
```

Send only 4 packets:

```bash
ping -c 4 ObjectiveIP
```

Example:

```bash
ping -c 4 10.0.0.10
```

### Purpose

Test if ICMP is reachable.

Suitable for initial determination:

```text
Basic network connectivity for target hosts

Is there any obvious missing package from the target?

Is the delay abnormal?
```

---

## Scenario 5: Ping Domain Name

### Command

```bash
ping -c 4 www.baidu.com
```

### Note

If ping domain name fails, there may be two types of issues:

```text
DNS Parsing failed

Objective IP Unattainable.
```

Can first resolve:

```bash
nslookup www.baidu.com
```

Or:

```bash
dig www.baidu.com
```

DNS-related content is organized separately in Chapter 07.

---

## Scenario 6: Ping Reachability Does Not Guarantee Service Port Reachability

`ping` only indicates:

```text
ICMP It's possible.
```

Does not indicate:

```text
TCP The port will be open.

HTTP The service must be normal.

Database connection must be normal.

Application protocol must be normal.
```

Example:

```text
ping 10.0.0.10 All
But... nc 10.0.0.10 3306 It's not working.
```

Indicates host is reachable, but database port may not be open or blocked.

---

## Scenario 7: Ping Unreachable Does Not Necessarily Mean Business Unreachable

Some environments disable ICMP, for example:

```text
Cloud security is off limits. ICMP

Firewall Ban ICMP

Network Device Ban ICMP

Target host is disabled for response ping
```

At this time `ping` is unreachable, but TCP service may be normal.

Should continue using:

```bash
nc -zv ObjectiveIP Port
```

Or:

```bash
curl -I http://Destination Address
```

---

## VI. nc: TCP Port Connectivity Troubleshooting

---

## Scenario 8: Test if Target Port is Reachable

### Command

```bash
nc -zv ObjectiveIP Port
```

Example:

```bash
nc -zv 10.0.0.5 3306
```

Test Kubernetes API Server:

```bash
nc -zv 10.0.0.10 6443
```

### Common Parameters

```text
-z
→ Check port only, do not send data

-v
→ Show Detailed Output
```

### Purpose

Test if TCP port can establish a connection.

Suitable for troubleshooting:

```text
Database port accessibility

Redis Port Availability

Kubernetes API Server Is it possible?

Nginx / Apply port open

Cross-host port connectivity
```

---

## Scenario 9: Set Timeout

### Command

```bash
nc -zv -w 2 ObjectiveIP Port
```

Example:

```bash
nc -zv -w 2 10.0.0.10 6443
```

### Common Parameters

```text
-w 2
→ Timeout 2 sec
```

Suitable for:

```text
Avoid being stuck for a long time.

Rapidly judge port connectivity in script

Batch check multiple ports
```

---

## Scenario 10: Common nc Result Interpretation

### Connection Success

```text
succeeded
open
connected
```

Explanation:

```text
TCP Port can create connections
```

### Connection Refused

```text
Connection refused
```

Common Causes:

```text
Target host available.

But the port is not listening.

Service not started

Service bugging at another address.

Our firewall refused.
```

### Connection Timeout

```text
Connection timed out
```

Common Causes:

```text
Network's out.

Security team not released.

Firewalls out.

Route abnormal.

No response.
```

---

## VII. curl: HTTP/HTTPS Service Testing

---

## Scenario 11: Access HTTP Service

### Command

```bash
curl http://Destination Address
```

Example:

```bash
curl http://10.0.0.5:8080
```

### Purpose

Test if HTTP service returns content normally.

---

## Scenario 12: Only View Response Headers

### Command

```bash
curl -I http://Destination Address
```

Example:

```bash
curl -I http://10.0.0.5:8080
```

### Common Parameters

```text
-I
→ Just watch it ring. Front
```

Suitable for quick confirmation:

```text
HTTP Status Code

Server Head

Is the response normal?

Whether or not to jump

Return 502 / 503 / 504
```

---

## Scenario 13: Detailed View Request Process

### Command

```bash
curl -v http://Destination Address
```

HTTPS:

```bash
curl -v https://Destination Address
```

Ignore certificate verification:

```bash
curl -vk https://Destination Address
```

### Common Parameters

```text
-v
→ Show detailed request process

-k
→ Ignore Certificate Validation
```

Suitable for troubleshooting:

```text
DNS Parsing Results

Connect to target IP and Port

TLS Shake hands.

Certification issues

HTTP Request and response header

Redirection process
```

---

## Scenario 14: Set Request Timeout

### Command

```bash
curl -m 5 http://Destination Address
```

### Common Parameters

```text
-m 5
→ Wait at most 5 sec
```

Suitable for:

```text
Interface detection

Script Testing

Avoid prolonged obstruction.
```

---

## Scenario 15: Follow Redirects

### Command

```bash
curl -L http://Destination Address
```

Or:

```bash
curl -L -I http://Destination Address
```

### Common Parameters

```text
-L
→ Follow 301 / 302 Redirect
```

---

## Scenario 16: Common curl Result Judgment

### HTTP 200

```text
Service return normal
```

### HTTP 301/302

```text
Redirection occurring.
```

Can add:

```bash
curl -L
```

Follow redirects.

### HTTP 401/403

```text
Questions of accreditation or authority
```

### HTTP 404

```text
Path does not exist or route rules do not match
```

### HTTP 500

```text
Backend Internal Error
```

### HTTP 502/503/504

```text
Gateway or upstream service anomalies
```

Common Direction:

```text
Nginx / Load Balance Forward Failed

Backend service not available

Backend is not listening

Backend Timeout

DNS / upstream Configure Error
```

---

## 8. ss: Listening Ports and Connection Troubleshooting

---

## Scenario 17: Check Listening Ports

### Command

```bash
ss -tunlp
```

### Parameter Explanation

```text
-t
→ TCP

-u
→ UDP

-n
→ Numeric display, do not resolve name

-l
→ Just listening.

-p
→ Show Process
```

### Purpose

Check which ports the current host is listening on, along with the corresponding processes.

Suitable for determining:

```text
Whether the service is actually listening to the client mouth

The wiretap is... 0.0.0.0 Still? 127.0.0.1

Which process is the port for?

Whether port is occupied by other processes
```

---

## Scenario 18: Check TCP Connections

### Command

```bash
ss -antp
```

### Parameter Explanation

```text
-a
→ All Connections

-n
→ Number Display

-t
→ TCP

-p
→ Show Process
```

Suitable for checking:

```text
What is it? TCP Connection

Connect Status

Connect Peer IP and Port

Correspond to current process
```

---

## Scenario 19: Check Socket Statistics

### Command

```bash
ss -s
```

### Purpose

Check current socket statistics.

Suitable for initial judgment:

```text
Is the number of connections abnormal?

TIME-WAIT Is it a lot?

TCP Whether the connection accumulates
```

---

## Scenario 20: Check Specific Port

### Command

```bash
ss -tunlp | grep 8080
```

Check 3306:

```bash
ss -tunlp | grep 3306
```

Check 6443:

```bash
ss -tunlp | grep 6443
```

---

## Scenario 21: Understanding Listening Addresses

If you see:

```text
0.0.0.0:8080
```

Indicates:

```text
Listen to all IPv4 Address
External machines usually go through the host. IP Visits
```

If you see:

```text
127.0.0.1:8080
```

Indicates:

```text
Just listen to the loop address.
External machines cannot access directly.
```

If you see:

```text
10.0.0.10:8080
```

Indicates:

```text
Listen only to designations IP
Other cards IP This may not work.
```

---

## 9. netstat: Common Port Checking Tool in Old Environments

---

## Scenario 22: Check Listening Ports

### Command

```bash
netstat -tunlp
```

### Purpose

Similar to `ss -tunlp`, used to check listening ports and processes.

Note:

```text
The new system is recommended. ss
The old system is still common. netstat
```

---

## Scenario 23: Check All Connections

### Command

```bash
netstat -anp
```

### Purpose

Check all network connections and processes.

---

## Scenario 24: What to Do if netstat is Not Installed

Ubuntu / Debian:

```bash
apt install -y net-tools
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y net-tools
```

Or:

```bash
dnf install -y net-tools
```

---

## 10. Routing Troubleshooting: ip route

---

## Scenario 25: Check Routing Table

### Command

```bash
ip route
```

### Purpose

Check the local routing table.

Focus on:

```text
default Default route

Target segment route

Gateway Address

Which card?

Whether multiple conflict routes exist
```

Common Output:

```text
default via 10.0.0.1 dev eth0
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.10
```

---

## Scenario 26: Check Which Route is Used to Access a Target IP

### Command

```bash
ip route get ObjectiveIP
```

Example:

```bash
ip route get 10.0.0.5
```

### Purpose

Check which route is used to access a target IP:

```text
Which card did you actually take?

Source IP Which one?

Who's next?

Whether or not to hit the expected route
```

Suitable for troubleshooting:

```text
Multinet Host

Multiple default route

Strategic route

Cross-net access anomaly

Source IP Selection does not match expectations
```

---

## Scenario 27: Common Manifestations of Routing Abnormalities

```text
You can access some of the links and not another one.

From A Present. B It doesn't work, but... B Present. A All

The multiple card machine is wrong.

Return path inconsistent

Accessor went through the wrong gateway.

Container grid conflict with inner grid

VPN Route covered business segment
```

Troubleshooting Commands:

```bash
ip route
```

```bash
ip route get ObjectiveIP
```

```bash
traceroute -n ObjectiveIP
```

---

## 11. Link Path Troubleshooting: traceroute and tracepath

---

## Scenario 28: Check Network Path

### Command

```bash
traceroute ObjectiveIP
```

Without resolving hostnames:

```bash
traceroute -n ObjectiveIP
```

### Common Parameters

```text
-n
→ Do not invert hostname, show directly IP
```

### Purpose

Check which hops the traffic takes from this machine to the target.

Suitable for troubleshooting:

```text
Cross-Network Link

Crossing room link.

Public Access Path

Cloud Network Path

Road route

Intermediate network device blocked
```

---

## Scenario 29: tracepath

### Command

```bash
tracepath ObjectiveIP
```

### Purpose

Similar to `traceroute`.

Some systems default to easier use, and it can also be used for basic path troubleshooting.

---

## Scenario 30: Incomplete traceroute Results Are Not Necessarily Abnormal

Sometimes traceroute shows:

```text
*
*
*
```

Possible reasons:

```text
Intermediate device does not reply ICMP / UDP TTL Overtime.

Firewall restrictions

The cloud network hides the midpoint

Target disabled associated sound Response
```

Therefore, traceroute should be combined with actual business access results for judgment.

---

## 12. Network I/O Troubleshooting: sar, iftop, ip -s link

---

## Scenario 31: sar to Check Network Card Traffic

### Command

```bash
sar -n DEV 1 5
```

### Parameter Explanation

```text
-n DEV
→ View card traffic

1 5
→ Sampling every second. 5 Minor
```

### Key Fields

```text
rxkB/s
→ Received traffic per second

txkB/s
→ Send traffic per second
```

### Purpose

Check real-time traffic of different network cards.

Suitable for determining:

```text
Which card traffic is high?

Whether to approach bandwidth ceiling

Is the flow high or high? High
```

---

## Scenario 32: What to Do if sar is Not Installed

`sar` comes from the `sysstat` package.

Ubuntu / Debian:

```bash
apt install -y sysstat
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

---

## Scenario 33: iftop to Check Who is Using Traffic

### Command

```bash
iftop
```

Without reverse resolving hostnames:

```bash
iftop -n
```

Specify network card:

```bash
iftop -i eth0
```

### Common Parameters

```text
-n
→ Do not reverse hostname

-i eth0
→ Assign a card
```

### Purpose

Check:

```text
Which one? IP Most communications with this machine

Where does the current flow come from?

Where does the current flow go?
```

Suitable for troubleshooting:

```text
Bandwidth anomaly

Unusual outreach

Big traffic transfer

Backup task full of networks

Interface Call Abnormal
```

---

## Scenario 34: What to Do if iftop is Not Installed

Ubuntu / Debian:

```bash
apt install -y iftop
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y iftop
```

Or:

```bash
dnf install -y iftop
```

---

## Scenario 35: ip -s link to Check Network Card Error Packets

### Command

```bash
ip -s link
```

Check specified network card:

```bash
ip -s link show eth0
```

### Purpose

Check packet statistics of the network card.

Focus on:

```text
RX errors

TX errors

RX dropped

TX dropped

overruns

carrier
```

Common Judgment:

```text
errors Growth
→ Possible links, drive, virtualization, cybercards, switchboard anomalies.

dropped Growth
→ Could be full, the system can't handle, network congestion, driving problems.
```

---

## 13. Continuous Packet Transmission and Joint Testing

---

## Scenario 36: Use ping for Continuous Packet Transmission

### Command

```bash
ping ObjectiveIP
```

### Purpose

Continuously send ICMP packets.

Suitable for pairing with packet capture on the opposite end:

```text
Confirm whether the request reaches the opposite end

Make sure the link's missing.

Confirms whether the network strategy is released
```

Stop:

```text
Ctrl + C
```

---

## Scenario 37: Specify ping Packet Size and Interval

### Command

```bash
ping -s 1400 -i 0.2 ObjectiveIP
```

### Parameter Explanation

```text
-s
→ Assign ICMP Package Size

-i
→ Packing interval
```

### Purpose

Suitable for troubleshooting:

```text
MTU Related issues

I lost the chain.

Network equipment ACL

Cloud Web Policy

Both ends grab the package.
```

Production Note:

```text
Don't deliver HF in a production environment for long periods.
Avoid affecting links and target hosts
```

---

## Scenario 38: Continuous Port Testing

### Command

```bash
while true; do nc -zv 10.0.0.5 3306; sleep 1; done
```

Set timeout:

```bash
while true; do nc -zv -w 2 10.0.0.5 3306; sleep 1; done
```

### Purpose

Continuously test whether a port is reachable.

Suitable for:

```text
Watch when port returns during service restart

Firewall rules adjusted for verification

Validation after release

The end grab bag confirmed the request.
```

---

## Scenario 39: Continuous HTTP Requests

### Command

```bash
while true; do curl -I http://10.0.0.5:8080; sleep 1; done
```

With timeout:

```bash
while true; do curl -m 3 -I http://10.0.0.5:8080; sleep 1; done
```

### Purpose

Continuously initiate HTTP requests.

Suitable for pairing with:

```text
Service access log

Service error log

tcpdump Grab the bag.

Nginx upstream Adjustment

Apply restart authentication
```

---

## 14. tcpdump: Packet Capture for Request and Response Verification

---

## Scenario 40: Capture Traffic on a Specific Network Card

### Command

```bash
tcpdump -i eth0
```

### Purpose

Capture network packets on a specific network card.

---

## Scenario 41: Do Not Resolve Hostnames and Port Names

### Command

```bash
tcpdump -i eth0 -nn
``` /think

### Parameter Description

```text
-nn
→ Do not parse host and port names, show them directly IP and Port
```

Production troubleshooting is recommended to add `-nn` to avoid DNS reverse lookup affecting speed.

---

## Scenario 42: Capture Specific Host

### Command

```bash
tcpdump -i eth0 -nn host 10.0.0.5
```

### Function

Capture traffic related to a specific IP address only.

---

## Scenario 43: Capture Specific Port

### Command

```bash
tcpdump -i eth0 -nn port 3306
```

Capture HTTP:

```bash
tcpdump -i eth0 -nn port 80
```

Capture HTTPS:

```bash
tcpdump -i eth0 -nn port 443
```

---

## Scenario 44: Capture Specific Host and Port

### Command

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306
```

### Function

Suitable for precise verification:

```text
Whether the flow of a host to a port has arrived

Response to request

Whether to create a connection
```

---

## Scenario 45: Capture All Network Interfaces

### Command

```bash
tcpdump -i any -nn
```

Specify port:

```bash
tcpdump -i any -nn port 8080
```

Specify host and port:

```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080
```

### Function

When unsure which network interface traffic flows through, using `-i any` is more convenient.

---

## Scenario 46: Exit After Capturing Fixed Number of Packets

### Command

```bash
tcpdump -i eth0 -nn -c 20
```

Specify condition:

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306 -c 20
```

### Parameter Description

```text
-c 20
→ Catch! 20 Automatically exit after a package
```

---

## Scenario 47: Save as pcap File

### Command

```bash
tcpdump -i eth0 -nn -w /tmp/test.pcap
```

Specify host and port:

```bash
tcpdump -i eth0 -nn host 10.0.0.5 and port 3306 -w /tmp/mysql-test.pcap
```

### Function

After saving, it can be analyzed using Wireshark.

Suitable for:

```text
Complex network issues

We need to give it to the network team.

Need to retain evidence

Needs View TCP Shake hands and repeat.
```

---

## Scenario 48: Common tcpdump Judgment

### Can see request packets, but not response packets

Possible direction:

```text
We've requested this machine, but it's not ringing. Response

This is the firewall.

Apply Unprocessed

The service is not wired.

Return path abnormal
```

### Can't see request packets

Possible direction:

```text
Request not made to the station.

Upstream firewall blocked.

Cloud Security Unit not released

There's no route.

Client requesting target error
```

### Has SYN, no SYN-ACK

Possible direction:

```text
Target port is not listening

Firewalls out.

No response from service

kernel not returned
```

### Has SYN, SYN-ACK, ACK

Explanation:

```text
TCP Three handshakes established
```

If the business still has issues, continue to check application layer protocols and logs.

---

## Fifteen, Typical Scenario Troubleshooting

---

## Scenario 49: Service Unreachable

### Troubleshooting Order

```text
ping

→ nc / curl

→ ss

→ Firewall / Security team

→ traceroute / ip route

→ tcpdump

→ Log
```

### Command

```bash
ping -c 4 ObjectiveIP
```

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
curl -I http://ObjectiveIP:Port
```

```bash
ss -tunlp | grep Port
```

```bash
ip route get ObjectiveIP
```

```bash
traceroute -n ObjectiveIP
```

```bash
tcpdump -i any -nn host ObjectiveIP and port Port
```

---

## Scenario 50: Local Service is Listening, but External Access is Unreachable

### Troubleshooting Direction

```text
Services listening 0.0.0.0

Is the firewall clear?

Is Cloud Security clear?

Is the route normal?

Whether to bind wrong IP

Is there any? NAT / Agent level issues
```

### Command

```bash
ss -tunlp | grep Port
```

```bash
ip a
```

```bash
iptables -L -n -v
```

```bash
ip route
```

```bash
tcpdump -i any -nn port Port
```

Focus on listening address:

```text
0.0.0.0:Port
→ Usually external access

127.0.0.1:Port
→ Visits only

AssignIP:Port
→ It's just a visit. IP It's possible.
```

---

## Scenario 51: Ping is Reachable but Port is Unreachable

### Possible Causes

```text
Service not started

Service does not listen to target end mouth

Services only listen. 127.0.0.1

Firewall interdiction TCP Port

Security team did not release port

Application anomaly

Port occupied by other services
```

### Command

```bash
ping -c 4 ObjectiveIP
```

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
ss -tunlp | grep Port
```

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

---

## Scenario 52: Port is Reachable but HTTP Returns 502 / 504

### Possible Causes

```text
Nginx Accessible, but backend upstream It's not working.

Backend service timeout

Backend service port anomaly

Proxy Configuration Error

Backend application processing slow

DNS or upstream Parsing anomaly
```

### Command

```bash
curl -I http://Destination Address
```

```bash
curl -v http://Destination Address
```

```bash
ss -tunlp
```

```bash
journalctl -u nginx -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

Continue testing backend:

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -I http://BackendIP:Backend
```

---

## Scenario 53: Network Traffic Abnormality

### Troubleshooting Order

```text
sar -n DEV

→ iftop -n

→ ss -s

→ ip -s link

→ tcpdump
```

### Command

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
ip -s link
```

```bash
tcpdump -i any -nn
```

---

## Scenario 54: Joint Debugging Network Connectivity

### Operation Order

```text
End ping / nc / curl Continuous flow

→ End tcpdump Grab the bag.

→ Compared to whether the request was received or returned Response

→ Join the firewall. / Route / Security team continues to locate.
```

### Local Continuous Testing

```bash
while true; do nc -zv -w 2 10.0.0.5 3306; sleep 1; done
```

Or:

```bash
while true; do curl -m 3 -I http://10.0.0.5:8080; sleep 1; done
```

### Remote Packet Capture

```bash
tcpdump -i any -nn host EndIP and port Destination Port
```

If remote side doesn't see packets:

```text
Problem between End and End
```

If remote side sees request but no response:

```text
Problem may be end-to-end service, firewall, application or return route
```

---

## Sixteen, Production Troubleshooting Notes

---

## 1. Do Not Rely Only on Ping

`ping` can only test ICMP.

Production troubleshooting should combine:

```bash
nc -zv ObjectiveIP Port
```

```bash
curl -I http://Destination Address
```

```bash
tcpdump
```

---

## 2. Port Reachable Does Not Mean Business is Normal

Port reachable only indicates TCP connection may be established.

Business may still fail at:

```text
HTTP Status code abnormal.

Authentication Failed

Certificate anomaly

Backend Timeout

Protocol does not match

Application Logical Error
```

So also check:

```bash
curl -v
```

```bash
journalctl -u Service Name
```

```bash
tail -f Apply Log
```

---

## 3. Capture Should Control Scope

In production environment, it's not recommended to directly capture for a long time:

```bash
tcpdump -i any
```

Recommended to add conditions:

```bash
tcpdump -i any -nn host ObjectiveIP and port Port -c 100
```

Or save file:

```bash
tcpdump -i any -nn host ObjectiveIP and port Port -w /tmp/test.pcap
```

---

## 4. Avoid Affecting Business When Troubleshooting Large Traffic

When using `iftop`, `tcpdump`, `sar` for troubleshooting, note:

```text
Don't take long to grab the full package.

Don't. pcap Write to a disk with insufficient space

Don't do high-flow measurements at the peak.

No large-scale scanning of production sites.
```

---

## 5. On Multi-NIC Hosts, Focus on Source IP and Outgoing Interface

When troubleshooting multi-NIC machines, must check:

```bash
ip route get ObjectiveIP
```

Confirm:

```text
Actual Source IP Which one?

Which card is the actual interface?

Did you take the expected gateway?
```

---

## Seventeen, Common Commands in This Article

---

## Network Interface and IP

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

## Basic Connectivity

```bash
ping ObjectiveIP
```

```bash
ping -c 4 ObjectiveIP
```

```bash
ping -s 1400 -i 0.2 ObjectiveIP
```

---

## Port Testing

```bash
nc -zv ObjectiveIP Port
```

```bash
nc -zv -w 2 ObjectiveIP Port
```

```bash
while true; do nc -zv -w 2 10.0.0.5 3306; sleep 1; done
```

---

## HTTP / HTTPS Testing

```bash
curl http://Destination Address
```

```bash
curl -I http://Destination Address
```

```bash
curl -v http://Destination Address
```

```bash
curl -vk https://Destination Address
```

```bash
curl -m 5 http://Destination Address
```

```bash
curl -L -I http://Destination Address
```

```bash
while true; do curl -m 3 -I http://10.0.0.5:8080; sleep 1; done
```

---

## Listening and Connection

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
ip route get ObjectiveIP
```

```bash
traceroute ObjectiveIP
```

```bash
traceroute -n ObjectiveIP
```

```bash
tracepath ObjectiveIP
```

---

## Network Traffic

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
tcpdump -i eth0 -nn -c 20
```

```bash
tcpdump -i eth0 -nn -w /tmp/test.pcap
```

---

## Tool Installation

Install net-tools on Ubuntu/Debian:

```bash
apt install -y net-tools
```

Install net-tools on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y net-tools
```

Or:

```bash
dnf install -y net-tools
```

Install sysstat on Ubuntu/Debian:

```bash
apt install -y sysstat
```

Install sysstat on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Install iftop on Ubuntu/Debian:

```bash
apt install -y iftop
```

Install iftop on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y iftop
```

Or:

```bash
dnf install -y iftop
```

Install traceroute on Ubuntu/Debian:

```bash
apt install -y traceroute
```

Install traceroute on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y traceroute
```

Or:

```bash
dnf install -y traceroute
```

Install tcpdump on Ubuntu/Debian:

```bash
apt install -y tcpdump
```

Install tcpdump on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y tcpdump
```

Or:

```bash
dnf install -y tcpdump
```

---

## Eighteen. One-Sentence Summary

The core of network troubleshooting isn't just about whether `ping` works, but:

```text
IP Correct?

→ Is the route correct?

→ Port listening

→ Port Availability

→ Normality of application protocol

→ Whether the traffic has arrived

→ Response Return
```

Service unreachability troubleshooting chain:

```text
ping

→ nc / curl

→ ss

→ Firewall / Security team

→ traceroute / ip route

→ tcpdump

→ Log
```

Network traffic anomaly troubleshooting chain:

```text
sar -n DEV

→ iftop -n

→ ss -s

→ ip -s link

→ tcpdump
```

Joint debugging network connectivity chain:

```text
Organisation ping / nc / curl

→ End tcpdump Grab the bag.

→ To determine whether the request has arrived

→ Judge whether the response returned

→ Continue to locate along routes, firewalls, security units, logs.
```

Production recommendation:

```text
Don't just use it. ping Adjudication of service availability
Don't make it sound like business.
You'll have to add it to the bag. host / port / -c Scope of limitations
Domino machines must see. ip route get
When external access is out, focus on the listening address, right? 127.0.0.1
Network barriers need to read both the machine, the other end, the middle link and the application log
```