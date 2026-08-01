# 16-Docker Networking: bridge, veth, iptables, and NAT

#Docker #DockerNetwork #bridge #docker0 #veth #iptables #NAT #PortMap #ContainerNetwork #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/16-Docker Networking: bridge, veth, iptables, and NAT.md

---

## I. Document Overview

This document explains the underlying mechanisms of Docker's default bridge network, focusing on:

- What is Docker's default bridge network
- What is `docker0`
- What is a veth pair
- How container network interfaces connect to the host
- Relationship between network namespace and container networking
- How container IPs are assigned
- How containers access the internet
- How external access reaches containers
- Underlying principles of Docker port mapping
- iptables DNAT/SNAT/MASQUERADE
- `DOCKER` chain and `POSTROUTING` chain
- Why container's `127.0.0.1` is not the host
- Why services only listening on `127.0.0.1` fail to be accessed
- Differences between bridge/host/none network modes
- User-defined bridge networks
- Docker DNS and service name resolution
- Common Docker networking troubleshooting

Goals:

- Understand Docker's default network path
→ Explain relationship between docker0/veth/namespace
→ Understand why NAT is needed for internet access
→ Understand why port mapping allows container access
→ Troubleshoot port issues, DNS anomalies, and subnet conflicts
→ Explain Docker networking issues from the underlying layer

---

## II. Docker Networking Overview

Docker's default network can be simplified as:

```text
Containers network namespace
→ Inside the container eth0
→ veth pair
→ Host vethxxx
→ docker0 bridge
→ Host physics card
→ External network
```

When external access reaches a container:

```text
External Client
→ Host IP:Host Port
→ iptables DNAT
→ Containers IP:Container Port
→ Container services
```

When a container accesses the internet:

```text
Containers IP
→ docker0
→ iptables SNAT / MASQUERADE
→ Host physics card IP
→ External network
```

One-sentence summary:

```text
docker0 Cover the bridge.
veth To connect the container to the host.
iptables Responsible NAT and Port Forward
network namespace Isolate container network view
```

---

## III. Docker Default Network Types

Check Docker networks:

```bash
docker network ls
```

Common default networks:

```text
bridge
host
none
```

Check default bridge:

```bash
docker network inspect bridge
```

---

## IV. Bridge Networking Mode

---

## Scenario 1: What is the default bridge network

If a container is started without specifying a network:

```bash
docker run -d nginx
```

Normally, the container joins the default bridge network.

The default bridge network is typically associated with the host's `docker0` bridge.

Check network:

```bash
docker network ls
```

Check bridge details:

```bash
docker network inspect bridge
```

Check host's docker0:

```bash
ip addr show docker0
```

Common addresses may be:

```text
172.17.0.1/16
```

Explanation:

```text
docker0
→ Docker Created on host Linux bridge
→ Default bridge The container in the network has access to the host network.
```

---

## Scenario 2: What is docker0

`docker0` is a Linux bridge on the host.

It can be understood as:

```text
docker0 = Virtual switch on the host.
```

When a container joins the default bridge network, it connects to docker0 via a veth pair.

Check docker0:

```bash
ip link show docker0
```

Check address:

```bash
ip addr show docker0
```

Check routing:

```bash
ip route
```

May see similar:

```text
172.17.0.0/16 dev docker0
```

Meaning:

```text
Visits 172.17.0.0/16 Network
→ Come on. docker0
```

---

## Scenario 3: What is a veth pair

A veth pair can be understood as:

```text
Two ends of a virtual web line.
```

One end in the container, usually called:

```text
eth0
```

The other end on the host, usually called:

```text
vethxxxx
```

Data flow:

```text
Containers eth0
→ veth pair
→ Host vethxxxx
→ docker0
```

Check host's veth:

```bash
ip link
```

Enter container to check network interface:

```bash
docker exec -it ContainersID ip addr
```

The container typically shows:

```text
lo
eth0
```

---

## Scenario 4: Container network namespace

Each regular container usually has its own network namespace.

Check network interface in container:

```bash
docker exec -it ContainersID ip addr
```

Check routing in container:

```bash
docker exec -it ContainersID ip route
```

Check DNS in container:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

Get container's main process PID on host:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Check process namespace:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/ns
```

Check network namespace:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/ns/net
```

Understanding:

```text
The container has its own network. namespace
So it has its own card.IP, route, port listening and loopback
```

---

## V. How Container IPs are Assigned

---

## Scenario 5: Check Container IP

Check container IP:

```bash
docker inspect ContainersID | grep IPAddress
```

More precisely:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ContainersID
```

Enter container to check:

```bash
docker exec -it ContainersID ip addr
```

---

## Scenario 6: Default bridge network IP

Common subnet for default bridge network:

```text
172.17.0.0/16
```

Common address for host's docker0:

```text
172.17.0.1
```

Container may receive:

```text
172.17.0.2
172.17.0.3
172.17.0.4
```

Check bridge network:

```bash
docker network inspect bridge
```

Focus on:

```text
IPAM
Containers
Gateway
Subnet
```

---

## Scenario 7: Container IP is not suitable as long-term access address

Container IPs may change.

For example:

```text
Rebuild container after removal
→ IP Possible changes

Packagings into different networks
→ IP Could be different.

Compose Project reconstruction
→ IP Possible changes
```

Business should not hardcode container IPs long-term.

Better recommendation:

```text
Docker Compose Use of service name
Kubernetes Use Service
External access using port map or Ingress / LB
```

---

## VI. Container Internet Access Path

---

## Scenario 8: Basic path for container internet access

When a container accesses the internet, the path is roughly:

```text
Container Process
→ Containers eth0
→ veth pair
→ docker0
→ Host route
→ iptables POSTROUTING MASQUERADE
→ Host physics card
→ External network
```

Which is:

```text
Private container IP
→ By NAT Sung-hoon Host IP
→ Access to external networks
```

This type of NAT is usually called:

```text
SNAT
MASQUERADE
```

---

## Scenario 9: Check host routing

```bash
ip route
```

May see:

```text
default via 10.0.0.1 dev eth0
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

Understanding:

```text
172.17.0.0/16
→ Docker bridge Network

default
→ Default route for host access to external networks
```

---

## Scenario 10: Check NAT table

Check NAT table:

```bash
iptables -t nat -L -n
```

Check more detailed rules:

```bash
iptables -t nat -L -n -v
```

Check POSTROUTING:

```bash
iptables -t nat -L POSTROUTING -n -v
```

May see similar MASQUERADE rules:

```text
MASQUERADE  all  --  172.17.0.0/16  !172.17.0.0/16
```

Meaning:

```text
From Docker bridge Network traffic
Visits Docker bridge On the segment
Do the source address disguise
```

Which converts container source IP to host's outgoing IP.

---

## Scenario 11: Troubleshoot container internet access failure

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Test IP connectivity:

```bash
ping 8.8.8.8
```

Test domain resolution:

```bash
nslookup www.baidu.com
```

Check container routing:

```bash
ip route
```

Check container DNS:

```bash
cat /etc/resolv.conf
```

Check host routing:

```bash
ip route
```

Check host NAT:

```bash
iptables -t nat -L -n -v
```

Check host forwarding parameters:

```bash
sysctl net.ipv4.ip_forward
```

Normal should allow forwarding:

```text
net.ipv4.ip_forward = 1
```

Temporarily enable:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

## VII. External Access to Container Path

---

## Scenario 12: Basic Port Mapping Linkage

Start container:

```bash
docker run -d -p 8080:80 nginx
```

Access:

```text
http://HostIP:8080
```

Underlying linkage can be understood as:

```text
External Client
→ Host IP:8080
→ iptables DNAT
→ Containers IP:80
→ Inside the container Nginx
```

Check port mapping:

```bash
docker ps
```

Or:

```bash
docker port ContainersID
```

Check host listening:

```bash
ss -tunlp | grep 8080
```

Check NAT rules:

```bash
iptables -t nat -L DOCKER -n -v
```

---

## Scenario 13: What is DNAT

DNAT can be understood as:

```text
Organisation
```

External access:

```text
HostIP:8080
```

Is converted into:

```text
ContainersIP:80
```

That is:

```text
Destination NAT
```

In Docker port mapping, DNAT is responsible for forwarding traffic accessing the host's port to the container's port.

---

## Scenario 14: Check DOCKER Chain

Check NAT table DOCKER chain:

```bash
iptables -t nat -L DOCKER -n -v
```

May see rules like:

```text
tcp dpt:8080 to:172.17.0.2:80
```

Meaning:

```text
Access host 8080 Port
→ Forward to Container 172.17.0.2:80
```

Check full NAT table:

```bash
iptables -t nat -S
```

Check DOCKER chain rules:

```bash
iptables -t nat -S DOCKER
```

---

## Scenario 15: Why Check Host Listening for Port Mapping

Start:

```bash
docker run -d -p 8080:80 nginx
```

Check:

```bash
ss -tunlp | grep 8080
```

If no listening is visible, possible reasons:

```text
Packaging failed.
Port map failed
Port conflicted
Docker iptables The rules are abnormal.
Docker daemon Unusual
```

Check container:

```bash
docker ps -a
```

Check logs:

```bash
docker logs ContainersID
```

Check port mapping:

```bash
docker port ContainersID
```

---

## Scenario 16: Port Mapping Bound to 127.0.0.1 Only

Start:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Meaning:

```text
Just listen to the host's address.
External machines cannot access
```

Check listening:

```bash
ss -tunlp | grep 8080
```

May see:

```text
127.0.0.1:8080
```

Host machine access:

```bash
curl -I http://127.0.0.1:8080
```

External machine access:

```text
http://HostIP:8080
```

Usually not reachable.

If you want external access:

```bash
docker run -d -p 8080:80 nginx
```

Or bind to specific host IP:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

---

## VIII. Misconceptions About 127.0.0.1 Inside Containers

---

## Scenario 17: Who is 127.0.0.1 Inside a Container

Inside container:

```text
127.0.0.1
```

Represents the container itself.

Host machine:

```text
127.0.0.1
```

Represents the host itself.

Another container:

```text
127.0.0.1
```

Represents another container itself.

Therefore:

```text
Containers A Visits 127.0.0.1
→ The container was visited. A For yourself.

Containers A Access container B
→ Can't write. 127.0.0.1
```

---

## Scenario 18: Container Accessing Host Services

If the host service listens on:

```text
127.0.0.1:3306
```

The container generally cannot access it via host's 127.0.0.1.

Check host listening:

```bash
ss -tunlp | grep 3306
```

If only see:

```text
127.0.0.1:3306
```

The container usually cannot directly access.

You can let the service listen on:

```text
0.0.0.0:3306
```

Or host's internal IP:

```text
10.0.0.10:3306
```

Container access to host's docker0 gateway:

```text
172.17.0.1
```

Container test:

```bash
curl http://172.17.0.1:Port
```

Or:

```bash
nc -vz 172.17.0.1 Port
```

---

## Scenario 19: host.docker.internal

In some Docker Desktop environments, you can use:

```text
host.docker.internal
```

To access the host.

In native Linux Docker environments, it may not be supported by default.

You can add:

```bash
docker run --add-host=host.docker.internal:host-gateway -it alpine /bin/sh
```

When running the container.

Inside container access:

```bash
ping host.docker.internal
```

Or:

```bash
curl http://host.docker.internal:Port
```

---

## IX. Differences Between bridge / host / none Modes

---

## Scenario 20: bridge Mode

Default mode:

```bash
docker run -d nginx
```

Explicitly specify:

```bash
docker run -d --network bridge nginx
```

Features:

```text
The container is independent. network namespace
The container has its own. IP
Pass. docker0 Access to host
External visits are usually required -p Port Map
Container access to the offline usually goes through. SNAT / MASQUERADE
```

Suitable for:

```text
Normal packaging running
Test of single or multiple containers
Default Docker scene
```

---

## Scenario 21: host Mode

Start:

```bash
docker run -d --network host nginx
```

Features:

```text
Container shared host network namespace
No separate container. IP
No need. -p Port Map
The container listening port is the host. mouth
Network isolation is declining.
```

Check listening:

```bash
ss -tunlp
```

Suitable for:

```text
Performance sensitive scene
Direct use of the host web inn
Partial monitoring / Network Component
```

Risks:

```text
Port Conflict
Reduced isolation
Larger exposure.
```

---

## Scenario 22: none Mode

Start:

```bash
docker run -it --network none alpine /bin/sh
```

Features:

```text
There's a container. network namespace
But no ordinary network connection.
Usually only lo
```

Check:

```bash
ip addr
```

Suitable for:

```text
Extremely isolated scene
Manual Configuration Network Experiment
Special security scene
```

---

## X. User-Defined bridge Networks

---

## Scenario 23: Why Recommend Custom bridge

Default bridge network is relatively basic.

User-defined bridge networks have several advantages:

```text
The container can be parsed by name.
It's better off.
You can specify a subnet.
It can better organize multi-container services.
Compose Default also creates project networks
```

Create network:

```bash
docker network create app-net
```

Run container:

```bash
docker run -d --name web --network app-net nginx
```

```bash
docker run -d --name redis --network app-net redis:7
```

---

## Scenario 24: Service Name Resolution in Custom bridge

Enter web:

```bash
docker exec -it web /bin/sh
```

Resolve redis:

```bash
getent hosts redis
```

Or:

```bash
ping redis
```

Note:

```text
It doesn't have to be in the mirror. ping
```

Business configuration can write:

```text
redis:6379
```

Instead of hardcoding container IPs.

---

## Scenario 25: Create Network with Subnet

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  app-net
```

Check:

```bash
docker network inspect app-net
```

Suitable for:

```text
Quit VPC / IDC / VPN Net conflicts
Clear planning Docker Chile
Multiple environmental isolation
```

---

## XI. Docker DNS and Service Name Resolution

---

## Scenario 26: Container DNS Configuration

Check container DNS:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

Default bridge and custom bridge DNS behaviors may differ.

In user-defined bridge networks, Docker typically provides built-in DNS resolution capabilities, allowing containers to resolve each other via container names or service names.

---

## Scenario 27: DNS Abnormality Troubleshooting

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Check DNS:

```bash
cat /etc/resolv.conf
```

Test domain:

```bash
nslookup www.baidu.com
```

Or:

```bash
getent hosts www.baidu.com
```

Test service name:

```bash
getent hosts redis
```

Check DNS on host:

```bash
cat /etc/resolv.conf
```

Docker daemon configure DNS:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Note:

```text
Modify Docker DNS After that, re-created packagings are usually required to be fully effective.
```

---

## XII. iptables and Docker Chains

---

## Scenario 28: Common Docker iptables Chains

Docker maintains some iptables rules.

Common chains include:

```text
DOCKER
DOCKER-USER
DOCKER-ISOLATION-STAGE-1
DOCKER-ISOLATION-STAGE-2
```

Check filter table:

```bash
iptables -L -n -v
```

Check nat table:

```bash
iptables -t nat -L -n -v
```

Check rules:

```bash
iptables -S
```

Check NAT rules:

```bash
iptables -t nat -S
```

---

## Scenario 29: DOCKER Chain

`DOCKER` Chain is common in NAT port forwarding rules.

Check:

```bash
iptables -t nat -L DOCKER -n -v
```

Function:

```text
Processing Docker Port Map Related DNAT
```

Example:

```text
Host 8080
→ Containers 172.17.0.2:80
```

---

## Scenario 30: DOCKER-USER Chain

`DOCKER-USER` Chains are commonly used for user-defined firewall rules.

Features:

```text
Docker Officially recommends user-defined filter rules to DOCKER-USER Chain
Avoid direct changes Docker Automatically maintained chain
```

View:

```bash
iptables -L DOCKER-USER -n -v
```

Example: Deny access from a specific source to container services:

```bash
iptables -I DOCKER-USER -s 192.168.1.100 -j DROP
```

Note:

```text
The scope of impact must be confirmed before the firewall rules are changed in the production environment
Avoid interruption of business flows
```

---

## Scenario 31: MASQUERADE Rules

View:

```bash
iptables -t nat -L POSTROUTING -n -v
```

Common purposes:

```text
When containers access the outside network, convert the source address to host for export Address
```

Understanding:

```text
Containers IP Private. bridge Network
The outside network usually doesn't know how to get back to the container. IP
That's why I need it. SNAT / MASQUERADE
```

---

## Thirteen. iptables and nftables Notes

---

## Scenario 32: iptables Might Be nft Backend

In newer Linux systems, the `iptables` command might use the nftables backend.

View:

```bash
iptables --version
```

You might see:

```text
iptables v1.x.x (nf_tables)
```

Or:

```text
iptables v1.x.x (legacy)
```

Meaning:

```text
nf_tables
→ iptables Command used nftables Backend

legacy
→ Traditional iptables Backend
```

When troubleshooting Docker networks, note which backend the system is actually using.

---

## Scenario 33: Do Not Arbitrarily Clear iptables

High-risk operations:

```bash
iptables -F
```

```bash
iptables -t nat -F
```

Risks:

```text
Docker Port map rule lost
Container access to the offline is abnormal.
Kubernetes Node network anomaly
Business visits interrupted
```

If rules are accidentally cleared, you may need to restart Docker to rebuild the rules:

```bash
systemctl restart docker
```

Note:

```text
Production environment restarted Docker Could affect containers
Be careful.
```

---

## Fourteen. Docker Network Segment Conflicts

---

## Scenario 34: Why Network Segment Conflicts Occur

Docker's default bridge network segment is commonly:

```text
172.17.0.0/16
```

If the company's internal network, cloud VPC, VPN, or other networks also use the same or overlapping network segments, you may encounter:

```text
Could not close temporary folder: %s
Wrong route.
Timeout for access to external services
NAT Behavioural anomalies
```

---

## Scenario 35: Troubleshoot Network Segment Conflicts

Check Docker bridge:

```bash
docker network inspect bridge
```

Check host routing:

```bash
ip route
```

Check container routing:

```bash
docker exec -it ContainersID ip route
```

Confirm internal network segment:

```text
IDC Network
Clouds VPC Network
VPN Network
Office segment
Kubernetes Pod Network
Kubernetes Service Network
```

---

## Scenario 36: Modify Docker's Default Network Segment

Edit:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "bip": "172.31.0.1/24",
  "default-address-pools": [
    {
      "base": "172.32.0.0/16",
      "size": 24
    }
  ]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Verify:

```bash
docker network inspect bridge
```

Note:

```text
Modify Docker The network will affect existing containers.
The production environment requires maintenance windows
```

---

## Fifteen. Common Fault Cases

---

## Case 1: Port Mapping but Access is Unavailable

Start:

```bash
docker run -d -p 8080:80 nginx
```

Troubleshoot:

```bash
docker ps
```

```bash
docker port ContainersID
```

```bash
ss -tunlp | grep 8080
```

```bash
iptables -t nat -L DOCKER -n -v
```

```bash
docker logs ContainersID
```

Common causes:

```text
The container is not running.
Port Conflict
Service does not listen to container end mouth
Firewall blocked.
Security team not released.
It's bound. 127.0.0.1
iptables The rules are abnormal.
```

---

## Case 2: Host Can Access, External Machines Cannot

Check if bound to 127.0.0.1:

```bash
ss -tunlp | grep 8080
```

If you see:

```text
127.0.0.1:8080
```

It means only the host machine can access.

When starting, change to:

```bash
docker run -d -p 8080:80 nginx
```

Or bind to a specific IP:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

---

## Case 3: Container Can Ping IP but Cannot Resolve Domains

Inside the container:

```bash
ping 8.8.8.8
```

Can pass.

However:

```bash
nslookup www.baidu.com
```

Fails.

Troubleshoot:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

```bash
cat /etc/resolv.conf
```

Configure Docker DNS:

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Verify after recreating the container.

---

## Case 4: Container Cannot Access the Internet

Troubleshoot the chain:

```bash
docker exec -it ContainersID ip route
```

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

```bash
ip route
```

```bash
sysctl net.ipv4.ip_forward
```

```bash
iptables -t nat -L POSTROUTING -n -v
```

```bash
docker network inspect bridge
```

Common causes:

```text
The host itself cannot access the outside.
DNS Failed
ip_forward Unopened
iptables MASQUERADE The rules are abnormal.
Firewall restrictions
Docker bridge Unusual
```

---

## Case 5: Container Cannot Access Host's 127.0.0.1

Misunderstanding:

```text
Inside the container 127.0.0.1 = Host
```

Correct understanding:

```text
Inside the container 127.0.0.1 = The container itself.
```

Solution direction:

```text
Use host intranet IP
Use docker0 Gateway IP
Service listening 0.0.0.0
Use as necessary host.docker.internal
```

Check docker0:

```bash
ip addr show docker0
```

---

## Case 6: Docker Network Segment Conflicts with Company Internal Network

Troubleshoot:

```bash
docker network inspect bridge
```

```bash
ip route
```

```bash
docker exec -it ContainersID ip route
```

Handling:

```text
Planning Docker Network
Modify bip / default-address-pools
Avoid IDC / VPC / VPN / K8s Network
```

---

## Sixteen. Common Commands Summary

---

## Docker Network Basics

View network:

```bash
docker network ls
```

View bridge:

```bash
docker network inspect bridge
```

Create custom network:

```bash
docker network create app-net
```

Specify network segment to create network:

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  app-net
```

Delete network:

```bash
docker network rm app-net
```

---

## Container Network Inspection

View container IP:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ContainersID
```

View IP inside container:

```bash
docker exec -it ContainersID ip addr
```

View routing inside container:

```bash
docker exec -it ContainersID ip route
```

View DNS inside container:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

---

## Host Network Inspection

View docker0:

```bash
ip addr show docker0
```

View network interface:

```bash
ip link
```

View routing:

```bash
ip route
```

View listening:

```bash
ss -tunlp
```

View forwarding:

§

## iptables Inspection

View filter table:

```bash
iptables -L -n -v
```

View nat table:

```bash
iptables -t nat -L -n -v
```

View DOCKER chain:

```bash
iptables -t nat -L DOCKER -n -v
```

View POSTROUTING:

```bash
iptables -t nat -L POSTROUTING -n -v
```

View DOCKER-USER:

```bash
iptables -L DOCKER-USER -n -v
```

View rules:

```bash
iptables -S
```

View NAT rules:

```bash
iptables -t nat -S
```

View iptables backend:

```bash
iptables --version
```

---

## Port Mapping

Standard port mapping:

```bash
docker run -d -p 8080:80 nginx
```

Bind only to localhost:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Bind to specific IP:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

View mapping:

```bash
docker port ContainersID
```

View container:

```bash
docker ps
```

---

## namespace Related

Get container PID:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

View namespace:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/ns
```

Enter container network namespace to view IP:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' ContainersID)
```

```bash
nsenter -t $PID -n ip addr
```

---

## DNS Troubleshooting

Container DNS:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

Domain resolution inside container:

```bash
docker exec -it ContainersID nslookup www.baidu.com
```

Use getent:

```bash
docker exec -it ContainersID getent hosts www.baidu.com
```

Docker daemon DNS configuration:

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"]
}
```

---

## Seventeen. One-Sentence Summary

Docker's default bridge network can be summarized as:

```text
network namespace
→ Block container network view

veth pair
→ Connect container to host

docker0 bridge
→ Virtual switch on the host.

iptables
→ Do port forwarding and NAT
```

Container access to the internet chain:

```text
Containers eth0
→ veth
→ docker0
→ Host route
→ iptables MASQUERADE
→ Host physics card
→ External network
```

External access to container chain:

```text
External Client
→ Host IP:Port
→ iptables DNAT
→ Containers IP:Container Port
→ Container services
```

Port mapping understanding:

```text
-p 8080:80
→ Host 8080 Forward to Container 80

-p 127.0.0.1:8080:80
→ Only host home access is allowed

-p 10.0.0.10:8080:80
→ Only the specified host is bound IP
```

127.0.0.1 Understanding:

```text
Host 127.0.0.1
→ The host himself.

Inside the container 127.0.0.1
→ The container itself.

Container access host service
→ Inside the host. IPI don't know.docker0 Gateway or host.docker.internal
```

Production Recommendations:

```text
Docker The network needs to be planned in advance, and it has to be avoided. IDC / VPC / VPN / K8s Net conflicts
Port mapping needs to be clearly bound, not brainless. 0.0.0.0
Do not write dead containers for communication between services. IP, prioritize use of service names or platform service discovery
Don't leave it empty. iptables
Check. Docker The network has to look inside the container, the host,iptablesI don't know.Docker network
host Network mode should be used carefully.
```