# 16-Docker Network Underlying Mechanisms: Bridge, Veth, iptables, and NAT

# Docker # Docker Networks # Bridge # docker0 # veth # iptables # NAT # Port Mapping # Container Networks # Operations and Maintenance # Troubleshooting

---

## Recommended Path

03-Container Technology/16-Docker Network Underlying Mechanisms: Bridge, Veth, iptables, and NAT.md

---

## I. Document Description

This article explains the underlying workings of Docker's default bridge network, focusing on:

- What Docker's default bridge network is
- What `docker0` is
- What a veth pair is
- How container network cards are connected to host network cards
- The relationship between network namespaces and container networks
- How container IP addresses are generated
- The pathways for containers to access the external network
- The pathways for external access to containers
- The underlying principles of Docker port mapping
- iptables DNAT, SNAT, and MASQUERADE
- The `DOCKER` chain and the `POSTROUTING` chain
- Why `127.0.0.1` inside a container is not the host machine
- Why setting a service to listen only on `127.0.0.1` can cause access failures
- The differences between bridge, host, and none network modes
- How to create custom bridge networks
- Docker DNS and service name resolution
- Common troubleshooting for Docker networks

The goal is:

To understand the default Docker network pathways

→ To be able to explain the relationship between docker0, veth, and namespaces

→ To understand why NAT is required for containers to access the external network

→ To understand how port mapping allows access to containers

→ To be able to troubleshoot issues such as blocked ports, DNS errors, and IP range conflicts

→ To be able to explain Docker network problems from a fundamental level

---

## II. Overview of Docker Networks

Docker's default network can be simplified as follows:

```text
Container network namespace
→ eth0 inside the container
→ veth pair
→ vethxxx on the host machine
→ docker0 bridge
→ Physical network card of the host machine
→ External network
```

When accessing a container from the outside:

```text
External client
→ Host IP: Host port
→ iptables DNAT
→ Container IP: Container port
→ Service inside the container
```

When a container accesses the external network:

```text
Container IP
→ docker0
→ iptables SNAT/MASQUERADE
→ Physical network card IP of the host machine
→ External network
```

In one sentence:

```text
docker0 is responsible for bridging
veth connects containers to the host machine
iptables handles NAT and port forwarding
network namespaces isolate container networks
```

---

## III. Docker Default Network Types

To view Docker networks:

```bash
docker network ls
```

Common default networks include:

```text
bridge
host
none
```

To view the default bridge:

```bash
docker network inspect bridge
```

---

## IV. Bridge Network Mode

---

## Scenario 1: What is the Default Bridge Network?

If a container is started without specifying a network:

```bash
docker run -d nginx
```

By default, the container will join the default bridge network.

The default bridge network is usually associated with the `docker0` bridge on the host machine.

To view networks:

```bash
docker network ls
```

To view details of the bridge:

```bash
docker network inspect bridge
```

To view the host's docker0:

```bash
ip addr show docker0
```

A common address might be:

```text
172.17.0.1/16
```

Explanation:

```text
docker0
→ A Linux bridge created by Docker on the host machine
→ Containers in the default bridge network use it to connect to the host network
```

---

## Scenario 2: What is docker0?

`docker0` is a Linux bridge on the host machine.

It can be thought of as:

```text
docker0 = A virtual switch on the host machine
```

When a container joins the default bridge network, it connects to `docker0` through a pair of veth devices.

To view docker0:

```bash
ip link show docker0
```

To view its address:

```bash
ip addr show docker0
```

To view routes:

```bash
ip route
```

You might see something like:

```text
172.17.0.0/16 dev docker0
```

Meaning:

```text
Access to the 172.17.0.0/16 network segment
→ Goes through docker0
```

---

## Scenario 3: What is a veth pair?

MASQUERADE all -- 172.17.0.0/16 !172.17.0.0/16

Meaning:

Traffic from the Docker bridge network segment
When accessing non-Docker bridge network segments
Source address masking is performed
In other words, the container's source IP is converted into the host's outgoing IP.

---

## Scenario 11: Troubleshooting Container Access to the External Network

Enter the container:

```bash
docker exec -it containerID /bin/sh
```

Test IP connectivity:

```bash
ping 8.8.8.8
```

Test domain name resolution:

```bash
nslookup www.baidu.com
```

View container routes:

```bash
ip route
```

View container DNS settings:

```bash
cat /etc/resolv.conf
```

View host routes:

```bash
ip route
```

View NAT settings on the host:

```bash
iptables -t nat -L -n -v
```

Check forwarding parameters on the host:

```bash
sysctl net.ipv4.ip_forward
```

It should normally allow forwarding:

```text
net.ipv4.ip_forward = 1
```

Temporarily enable it:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

## Section 7: External Access to Containers

---

## Scenario 12: Basic Port Mapping Links

Start the container:

```bash
docker run -d -p 8080:80 nginx
```

Access it:

```text
http://hostIP:8080
```

The underlying links can be understood as follows:

```text
External client
→ Host IP:8080
→ iptables DNAT
→ Container IP:80
→ Nginx inside the container
```

View port mapping:

```bash
docker ps
```

Or:

```bash
docker port containerID
```

View host listening ports:

```bash
ss -tunlp | grep 8080
```

View NAT rules:

```bash
iptables -t nat -L DOCKER -n -v
```

---

## Scenario 13: What is DNAT?

DNAT can be understood as:

```text
Destination Address Translation
```

When accessing from the outside:

```text
Host IP:8080
```

It is converted to:

```text
Container IP:80
```

In other words, it's:

```text
Destination NAT
```

In Docker port mapping, DNAT is responsible for forwarding traffic accessing the host's ports to the container's ports.

---

## Scenario 14: Viewing DOCKER Chains

View the NAT table for the DOCKER chain:

```bash
iptables -t nat -L DOCKER -n -v
```

You may see rules like this:

```text
tcp dpt:8080 to:172.17.0.2:80
```

Meaning:

```text
Access to host port 8080
→ Forwarded to container port 172.17.0.2:80
```

View the complete NAT table:

```bash
iptables -t nat -S
```

View DOCKER chain rules:

```bash
iptables -t nat -S DOCKER
```

---

## Scenario 15: Why Check Host Listening Ports for Port Mapping?

Start the container:

```bash
docker run -d -p 8080:80 nginx
```

View listening ports:

```bash
ss -tunlp | grep 8080
```

If no listening port is found, possible reasons include:

```text
The container did not start successfully.
Port mapping failed.
The port is already in use.
Abnormal Docker iptables rules.
Abnormal Docker daemon.
```

View the container:

```bash
docker ps -a
```

Check logs:

```bash
docker logs containerID
```

View port mapping settings:

```bash
docker port containerID
```

---

## Scenario 16: Port Mapping That Binds Only to 127.0.0.1

Start the container:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Meaning:

```text
It only listens on the host's local loopback address.
External machines cannot access it.
```

View listening ports:

```bash
ss -tunlp | grep 8080
```

You may see:

```text
127.0.0.1:8080
```

Access from the host:

```bash
curl -I http://127.0.0.1:8080
``## Scenario 25: Creating a Network with a Specific Subnet

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  app-net
```

View:

```bash
docker network inspect app-net
```

Suitable for:

```text
Avoiding VPC / IDC / VPN address segment conflicts
Clearly planning Docker network addresses
Isolating multiple environments
```

---

## Chapter Eleven: Docker DNS and Service Name Resolution

---

## Scenario 26: Configuring Container DNS

View container DNS settings:

```bash
docker exec -it containerID cat /etc/resolv.conf
```

The DNS behavior may differ between default bridges and custom bridges.

In user-defined bridge networks, Docker typically provides built-in DNS resolution capabilities, allowing containers to resolve each other by name or service name.

---

## Scenario 27: Troubleshooting DNS Issues

Enter the container:

```bash
docker exec -it containerID /bin/sh
```

View DNS settings:

```bash
cat /etc/resolv.conf
```

Test domain names:

```bash
nslookup www.baidu.com
```

Or:

```bash
getent hosts www.baidu.com
```

Test service names:

```bash
getent hosts redis
```

View DNS settings on the host machine:

```bash
cat /etc/resolv.conf
```

Configure Docker daemon DNS:

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
After modifying Docker DNS settings, you usually need to recreate containers for the changes to take effect completely.
```

---

## Chapter Twelve: iptables and Docker Chains

---

## Scenario 28: Common Docker iptables Chains

Docker maintains several iptables rules.

Common chains include:

```text
DOCKER
DOCKER-USER
DOCKER-ISOLATION-STAGE-1
DOCKER-ISOLATION-STAGE-2
```

View the filter table:

```bash
iptables -L -n -v
```

View the nat table:

```bash
iptables -t nat -L -n -v
```

View rules:

```bash
iptables -S
```

View NAT rules:

```bash
iptables -t nat -S
```

---

## Scenario 29: THE DOCKER CHAIN

The `DOCKER` chain is commonly used for NAT port forwarding rules.

View:

```bash
iptables -t nat -L DOCKER -n -v
```

Function:

```text
Handles DNAT related to Docker port mapping
```

Example:

```text
Host machine 8080
→ Container 172.17.0.2:80
```

---

## Scenario 30: THE DOCKER-USER CHAIN

The `DOCKER-USER` chain is often used for custom firewall rules.

Feature:

```text
Docker officially recommends placing user-defined filtering rules in the DOCKER-USER chain
to avoid directly altering Docker's automatically maintained chains
```

View:

```bash
iptables -L DOCKER-USER -n -v
```

Example: Blocking access from a specific source to container services:

```bash
iptables -I DOCKER-USER -s 192.168.1.100 -j DROP
```

Note:

```text
Before modifying firewall rules in a production environment, you must confirm the potential impact
to avoid accidentally blocking business traffic
```

---

## Scenario 31: MASQUERADE Rules

View:

```bash
iptables -t nat -L POSTROUTING -n -v
```

Common function:

```text
When a container accesses the external network, it converts the source address to the host machine's outbound address
```

Understanding:

```text
Container IP addresses are within private bridge networks
External networks usually do not know how to reach these container IPs
Therefore, SNAT / MASQUERADE is required
```

---

## Chapter Thirteen: Notes on iptables and nftables

---

## Scenario 32: iptables May Be Using nftables as a Backend

In newer Linux systems, the `iptables` command may use nftables as its backend.

Check:

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
→ The iptables command uses nftables as a backend

legacy
→ Traditional iptables backend
```

When troubleshooting Docker networks, pay```bash
iptables -t nat -L POSTROUTING -n -v
```

```bash
docker network inspect bridge
```

Common causes:

```text
The host itself cannot access the external internet.
DNS failure.
ip_forward is not enabled.
Abnormal iptables MASQUERADE rules.
Firewall restrictions.
Docker bridge issues.
```

---

## Case 5: The container fails to access the host's 127.0.0.1

Misunderstanding:

```text
127.0.0.1 inside the container = the host.
```

Correct understanding:

```text
127.0.0.1 inside the container = the container itself.
```

Solutions:

```text
Use the host's internal IP address.
Use the docker0 gateway IP address.
Have services listen on 0.0.0.0.
Use host.docker.internal if necessary.
```

View docker0:

```bash
ip addr show docker0
```

---

## Case 6: The Docker network segment conflicts with the company's internal network

Troubleshooting:

```bash
docker network inspect bridge
```

```bash
ip route
```

```bash
docker exec -it container_id ip route
```

Solutions:

```text
Plan the Docker network segment carefully.
Modify the /bin/iproute2.conf file or use the bip / default-address-pools configuration.
Avoid using IP segments that overlap with those of the IDC, VPC, VPN, or K8s networks.
```

---

## XVI. Summary of Common Commands

---

## Basics of Docker Networking

View networks:

```bash
docker network ls
```

View the bridge:

```bash
docker network inspect bridge
```

Create a custom network:

```bash
docker network create app-net
```

Create a network with a specified IP range:

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  app-net
```

Delete a network:

```bash
docker network rm app-net
```

---

## Viewing Container Networks

View the container's IP address:

```bash
docker inspect -f "{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_id
```

View the IP addresses inside the container:

```bash
docker exec -it container_id ip addr
```

View the routing inside the container:

```bash
docker exec -it container_id ip route
```

View the DNS settings inside the container:

```bash
docker exec -it container_id cat /etc/resolv.conf
```

---

## Viewing the Host's Network

View docker0:

```bash
ip addr show docker0
```

View network interfaces:

```bash
ip link
```

View routing:

```bash
ip route
```

View listening ports:

```bash
ss -tunlp
```

View forwarding settings:

```bash
sysctl net.ipv4.ip_forward
```

---

## Viewing iptables

View the filter table:

```bash
iptables -L -n -v
```

View the nat table:

```bash
iptables -t nat -L -n -v
```

View the DOCKER chain:

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

View the iptables backend:

```bash
iptables --version
```

---

## Port Mapping

Map a regular port:

```bash
docker run -d -p 8080:80 nginx
```

Map to the local host only:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Map to a specific IP address:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

View the mapping:

```bash
docker port container_id
```

List containers:

```bash
docker ps
```

---

## Namespace-related Operations

Get the container's PID:

```bash
docker inspect -f "{{.State.Pid}}' container_id
```

View the namespace:

```bash
ls -l /proc/$(docker inspect -f "{{.State.Pid}}' container_id)/ns
```

Enter a container network namespace to view IP addresses:

```bash
PID=$(docker inspect -f "{{.State.Pid}}' container_id)
```

```bash
nsenter -