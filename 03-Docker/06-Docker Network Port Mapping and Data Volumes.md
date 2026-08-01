# 06-Docker Networking, Port Mapping, and Data Volumes

#Docker #Network #PortMap #DataVolume #volume #bindmount #bridge #HostNetwork #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/06-Docker Networking, Port Mapping, and Data Volumes.md

---

## I. Document Overview

This document organizes common operational capabilities related to Docker networking, port mapping, and data volumes, with focus on:

- View Docker Networks
- View Container IP
- Container Subnet Conflict Diagnosis with VPC / Internal Network
- Modify Docker Default Subnet
- Host Network Mode
- Standard Port Mapping
- Local Host-Only Access
- Bind to Specific Host IP
- View Port Mapping
- Create Custom Network
- Specify Subnet for Network
- Specify Network Driver Type
- View and Delete Networks
- Container Join/Leave Network
- Create Data Volumes
- View and Delete Data Volumes
- Mount Data Volumes
- Bind Host Directory
- View Container Mount Information

Goals:

- Understand Docker bridge network
→ View container IP and routing
→ Diagnose network conflicts with internal network segments
→ Correctly use port mapping
→ Create custom networks
→ Use volume and bind mount to manage data

---

## II. Docker Network Management

---

## Scenario 31: View Docker Networks

### Command

View Docker Networks:

```bash
docker network ls
```

View detailed information:

```bash
docker network inspect bridge
```

### Notes

`docker network ls` is used to view existing Docker networks.

Common networks include:

```text
bridge
host
none
```

Among them `bridge` is the default network, and ordinary containers typically join the default `bridge` network if not specified.

---

## Scenario 32: View Container IP

### Command

```bash
docker inspect ContainersID | grep IPAddress
```

More commonly used shorthand:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ContainersID
```

### Notes

Container IP is typically used for troubleshooting Docker network issues on the host.

Notes:

- In default bridge network, container IP is usually only valid on the host or within the same Docker network
- Not recommended to use container IP as a long-term stable address
- Production access prefers port mapping, reverse proxy, or orchestration platform Service

---

## Scenario 33: Container Subnet Conflict Diagnosis with VPC / Internal Network

### Problem Manifestations

- Container access to certain internal resources fails
- Routing anomalies
- Cloud VPC overlapping with Docker default subnet
- NAT / forwarding behavior anomalies

### Common Diagnosis Commands

View Docker Networks:

```bash
docker network inspect bridge
```

View host routing:

```bash
ip route
```

View container routing:

```bash
docker exec -it ContainersID ip route
```

### Notes

This is a very typical production issue, especially when Docker default bridge subnet overlaps with cloud VPC, self-built IDC subnet, or container default bridge subnet.

For example, when Docker default subnet overlaps with company internal network, cloud VPC, or VPN subnet, it may cause abnormal access to certain addresses.

Diagnosis should simultaneously focus on:

- Docker bridge subnet
- Host routing
- Container routing
- VPC / IDC subnet planning
- VPN subnet
- Security group / firewall rules

---

## Scenario 34: Modify Docker Default Subnet

### Configuration File

```bash
/etc/docker/daemon.json
```

### Example

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

### Notes

- `bip`: Specify default bridge network interface address
- `default-address-pools`: Specify subsequent custom network address pool
- Used to avoid conflicts with VPC / internal network subnets

### Effectiveness

After modification, Docker typically needs to be restarted:

```bash
systemctl restart docker
```

### Notes

Modifying Docker network configuration may affect existing container networks.

Before production modification, confirm:

- Whether existing containers depend on default bridge network
- Whether business is running
- Whether maintenance window is needed
- Whether Docker configuration needs to be backed up
- Whether containers or networks need to be recreated

---

## Scenario 35: Host Network Mode

### Command

```bash
docker run -d --network host nginx
```

### Meaning

- Container uses host network stack directly
- No port mapping is performed

### Notes

- Suitable for performance-sensitive scenarios with clear port control
- But reduces isolation

### Operations Understanding

In host network mode, containers no longer have independent network namespaces.

It can be understood as:

```text
Container process directly using host network
```

Therefore:

- No need for `-p` port mapping
- Container listening port is the host port
- Higher risk of port conflicts
- Weaker network isolation
- Diagnosis requires checking host port listening directly

Check listening ports:

```bash
ss -tunlp
```

---

## III. Docker Port Mapping

---

## Scenario 51: Standard Port Mapping

### Command

```bash
docker run -d -p 8080:80 nginx
```

### Meaning

- Host 8080 maps to container 80
- Binds to all host network interfaces by default

Equivalent understanding:

```text
0.0.0.0:8080 -> Containers:80
```

### Notes

This is the most common port mapping method.

Access method is typically:

```text
http://HostIP:8080
```

---

## Scenario 52: Local Host-Only Access

### Command

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

### Meaning

- Listens only on loopback address
- External machines cannot access directly

### Applicable Scenarios

- Local testing
- Only for local Nginx reverse proxy calls
- Not wanting to expose service directly

### Notes

This binds only to host `127.0.0.1`.

In other words:

```text
Host home access
External machines cannot access directly.
```

Suitable for services only intended to be accessed by local proxies.

---

## Scenario 53: Bind to Specific Host IP

### Command

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

### Meaning

- Binds to specific host IP
- Very useful in multi-network interface scenarios

### Notes

When host has multiple network interfaces or IPs, can bind to specific IP.

For example:

- Management network IP
- Business network IP
- Internal network IP
- External network IP

This reduces unnecessary exposure.

---

## Scenario 54: View Port Mapping

### Command

```bash
docker ps
```

Or:

```bash
docker port ContainersID
```

### Notes

`docker ps` can directly show container port mapping relationships.

For example:

```text
0.0.0.0:8080->80/tcp
```

`docker port` is better for detailed container port mapping.

---

## IV. Docker Networks and Data Volumes

---

## Scenario 65: Create Custom Network

### Command

```bash
docker network create my-net
```

### Notes

Custom networks are commonly used for communication between multiple containers.

Compared to default `bridge` network, custom bridge networks are typically more suitable for business container interconnection.

Common uses:

- Application container connects to database container
- Nginx reverse proxies backend services
- Multiple containers forming a small service environment
- Docker Compose automatically creates project network

---

## Scenario 66: Create Network with Specified Subnet

### Command /think

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-net
```

### Notes

- Used to avoid conflicts with VPC/internal network segments
- Common in production environment planning

### Operations Understanding

If the default Docker network segment conflicts with existing network, you can explicitly specify the network segment when creating the network.

Plan to avoid:

- IDC internal network segment
- Cloud VPC network segment
- VPN network segment
- Kubernetes Pod network segment
- Kubernetes Service network segment
- Other Docker host network segments

---

## Scenario 67: Specify Driver Type

### Command

```bash
docker network create \
  --driver bridge \
  my-net
```

### Common Drivers

```text
bridge   (Default, IC)
host     (Shared host network)
none     (no network)
```

### Notes

Common driver understanding:

- `bridge`: Default mode, suitable for single-machine Docker container communication
- `host`: Container uses host network directly
- `none`: Container has no network, suitable for special isolation scenarios

---

## Scenario 68: View and Delete Network

View network:

```bash
docker network ls
```

View details:

```bash
docker network inspect my-net
```

Delete network:

```bash
docker network rm my-net
```

### Notes

Before deleting a network, confirm that no containers are using it.

If a network is being used by containers, deletion will fail.

You can first check network details:

```bash
docker network inspect my-net
```

---

## Scenario 69: Container Join and Leave Network

Join during runtime:

```bash
docker run -d --network my-net nginx
```

Existing container join:

```bash
docker network connect my-net ContainersID
```

Leave network:

```bash
docker network disconnect my-net ContainersID
```

### Notes

Containers can specify network at startup or join a network later.

Common uses:

- Temporarily join test network
- Troubleshoot container communication
- Connect existing containers to new network
- Remove container from certain network

---

## Scenario 70: Create Data Volume

### Command

```bash
docker volume create my-volume
```

### Notes

Docker volume is a data volume managed by Docker.

Common uses:

- Persist data
- Avoid data loss when containers are deleted
- Save database data
- Save application uploaded files
- Save data reusable across containers

---

## Scenario 71: View and Delete Data Volume

View:

```bash
docker volume ls
```

Details:

```bash
docker volume inspect my-volume
```

Delete:

```bash
docker volume rm my-volume
```

### Notes

Confirm data needs before deleting a data volume.

Especially for volumes containing database, uploaded files, and business data, deletion must be cautious.

---

## Scenario 72: Mount Data Volume

### Command

```bash
docker run -d \
  -v my-volume:/data \
  nginx
```

### Notes

Meaning:

```text
my-volume -> Inside the container /data
```

Data in volume remains even after container deletion.

---

## Scenario 73: Bind Host Directory

### Command

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html \
  nginx
```

### Notes

- volume: Managed by Docker, recommended
- bind mount: Flexible, suitable for development and debugging

### Operations Understanding

Difference between volume and bind mount:

```text
volume
→ Docker Manage Paths
→ More suitable for production sustainability
→ Life cycle relative independent of container

bind mount
→ Mount host directory directly
→ Clear Path
→ Fit to debug, develop, configure mounted
→ More dependent on the host directory
```

---

## Scenario 74: View Container Mount Information

```bash
docker inspect ContainersID | grep Mounts -A 20
```

### Notes

This command is used to view container mount information.

Helps confirm:

- Whether volume is mounted
- Whether host directory is mounted
- Container mount path
- Host source path
- Whether mount is as expected

---

## Five. Common Troubleshooting Approaches

---

## 1. Troubleshoot Port Unreachability

First check if container is running:

```bash
docker ps
```

Check port mapping:

```bash
docker port ContainersID
```

Check host listening:

```bash
ss -tunlp
```

If using regular port mapping:

```bash
docker run -d -p 8080:80 nginx
```

Confirm host is listening on 8080.

If using local binding:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

External machines cannot access directly, which is expected behavior.

---

## 2. Troubleshoot Container Network Unreachability

Check Docker network:

```bash
docker network ls
```

Check network details:

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

Common causes:

- Container not joined correct network
- Host routing anomaly
- Docker bridge network segment conflict with internal network
- Firewall rule anomaly
- Target service not listening
- Security group not allowing traffic

---

## 3. Troubleshoot Data Volume Mount Anomalies

Check container details:

```bash
docker inspect ContainersID
```

Check mount information:

```bash
docker inspect ContainersID | grep Mounts -A 20
```

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect my-volume
```

If bind mount, check host path:

```bash
ls -ld /data/nginx
```

Common causes:

- Host directory not exists
- Host directory permission incorrect
- Container path written wrong
- Volume name written wrong
- SELinux/AppArmor security policy impact
- Container process user lacks read/write permission

---

## Six. Production Notes

---

## 1. Control Exposed Port Range for Port Mapping

Normal writing:

```bash
docker run -d -p 8080:80 nginx
```

Equivalent to:

```text
0.0.0.0:8080 -> Containers:80
```

This means all network interfaces may listen on this port.

If only local access is desired, recommend using:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

If only binding to certain internal IP, recommend using:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

---

## 2. Plan Docker Network Segment in Advance

Production environment should avoid Docker network segment conflicts with:

- Cloud VPC network segment
- IDC internal network segment
- VPN network segment
- Kubernetes Pod network segment
- Kubernetes Service network segment
- Office network segment
- Other Docker host network segments

Check Docker bridge network:

```bash
docker network inspect bridge
```

Check host routing:

```bash
ip route
```

---

## 3. Use host Network Mode with Caution

host network mode:

```bash
docker run -d --network host nginx
```

Advantages:

- More direct network path
- No need for port mapping
- Lower performance loss in some scenarios

Risks:

- Reduced network isolation
- Container ports directly occupy host ports
- Easy to port conflicts
- Larger exposure surface
- Confusion between host processes and container processes during troubleshooting

---

## 4. Confirm Data Value Before Deleting Volume

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect my-volume
```

Delete volume:

```bash
docker volume rm my-volume
```

In production environments, volume may contain:

- Database data
- Application uploaded files
- Configuration files
- Cache data
- Business operation data

Confirm data can be discarded before deletion.

---

## 5. Pay Attention to Host Directory Permissions for Bind Mount

Bind host directory:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html \
  nginx
```

Confirm host directory exists:

```bash
ls -ld /data/nginx
```

Create directory if needed:

```bash
mkdir -p /data/nginx
```

Check directory permissions and owner if container process cannot read/write.

---

## Seven. Common Commands Summary

---

## Docker Network

Check Docker network:

```bash
docker network ls
```

Check default bridge network details: /think

```bash
docker network inspect bridge
```

View details of a specified network:

```bash
docker network inspect my-net
```

Create a custom network:

```bash
docker network create my-net
```

Specify a subnet to create a network:

```bash
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-net
```

Specify a network driver:

```bash
docker network create \
  --driver bridge \
  my-net
```

Delete a network:

```bash
docker network rm my-net
```

Add a container to a network at startup:

```bash
docker run -d --network my-net nginx
```

Add an existing container to a network:

```bash
docker network connect my-net ContainersID
```

Remove a container from a network:

```bash
docker network disconnect my-net ContainersID
```

---

## Container IP and Routing

View container IP:

```bash
docker inspect ContainersID | grep IPAddress
```

Briefly view container IP:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ContainersID
```

View host routing:

```bash
ip route
```

View container internal routing:

```bash
docker exec -it ContainersID ip route
```

---

## Port Mapping

Standard port mapping:

```bash
docker run -d -p 8080:80 nginx
```

Only accessible locally on the host:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Bind to a specific host IP:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

View port mapping:

```bash
docker ps
```

View port mapping for a specific container:

```bash
docker port ContainersID
```

View host listening ports:

```bash
ss -tunlp
```

---

## host Network Mode

Use host network mode:

```bash
docker run -d --network host nginx
```

View host port listening:

```bash
ss -tunlp
```

---

## Data Volumes

Create a data volume:

```bash
docker volume create my-volume
```

View data volumes:

```bash
docker volume ls
```

View data volume details:

```bash
docker volume inspect my-volume
```

Delete a data volume:

```bash
docker volume rm my-volume
```

Mount a data volume:

```bash
docker run -d \
  -v my-volume:/data \
  nginx
```

Bind host directory:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html \
  nginx
```

View container mount information:

```bash
docker inspect ContainersID | grep Mounts -A 20
```

View host directory:

```bash
ls -ld /data/nginx
```

Create host directory:

```bash
mkdir -p /data/nginx
```

---

## VIII. One-Sentence Summary

The core of Docker networks, ports, and data volumes is:

Network determines how containers communicate

→ Port mapping determines how external access containers

→ Data volumes determine how container data is persisted

Network troubleshooting core path:

```text
docker network ls
→ docker network inspect
→ docker inspect View container network
→ ip route
→ Inside the container ip route
→ Check web conflict, firewall, security team
```

Port troubleshooting core path:

```text
docker ps
→ docker port ContainersID
→ ss -tunlp
→ Confirm binding address
→ Confirm the firewall. / Security team
→ Confirm that the service in the container is listening.
```

Data volume troubleshooting core path:

```text
docker volume ls
→ docker volume inspect
→ docker inspect View Mounts
→ Check host directory
→ Inspection Permissions
→ Check inside the container.
```

Production recommendations:

```text
Docker The network needs to be planned in advance, and it has to be avoided. VPC / IDC / VPN / K8s Net conflicts
Port map to control exposure. Do not default anything. 0.0.0.0
host Network mode should be used carefully.
volume Fit for production sustainability
bind mount scenes suitable for debugging, configuring mounts and specifying host paths
Delete volume We have to make sure the data is discarded.
```