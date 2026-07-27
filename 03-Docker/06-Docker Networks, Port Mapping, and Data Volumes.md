# 06-Docker Networks, Port Mapping, and Data Volumes

#Docker #Networks #Port Mapping #DataVolumes #volume #bindmount #bridge #host Network #Operation and Maintenance #Troubleshooting

---

## Recommended Path

03-Container Technology/06-Docker Networks, Port Mapping, and Data Volumes.md

---

## I. Document Description

This document compiles common operation and maintenance skills related to Docker networks, port mapping, and data volumes, focusing on the following aspects:

- Viewing Docker networks
- Checking container IP addresses
- Troubleshooting conflicts between container IP ranges and VPCs/intranets
- Modifying the default Docker network range
- Host network mode
- General port mapping
- Access limited to the host machine only
- Binding to a specific host IP address only
- Viewing port mappings
- Creating custom networks
- Creating networks with specified IP ranges
- Specifying network driver types
- Viewing and deleting networks
- Adding and removing containers from networks
- Creating data volumes
- Viewing and deleting data volumes
- Mounting data volumes
- Binding to host directories
- Checking container mount information

The goal is to:

- Understand Docker bridge networks
- Be able to view container IP addresses and routing tables
- Troubleshoot conflicts between container networks and intranet IP ranges
- Use port mapping correctly
- Create custom networks
- Manage data using volumes and bind mounts

---

## II. Docker Network Management

---

## Scenario 31: Viewing Docker Networks

### Command

To view existing Docker networks:

```bash
docker network ls
```

For detailed information about a specific network:

```bash
docker network inspect bridge
```

### Explanation

`docker network ls` lists all available networks on the current Docker instance.

Common network types include:

```text
bridge
host
none
```

By default, new containers join the `bridge` network unless otherwise specified.

---

## Scenario 32: Checking Container IP Addresses

### Command

To view a container's IP address:

```bash
docker inspect containerID | grep IPAddress
```

A more concise alternative:

```bash
docker inspect -f "{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' containerID
```

### Explanation

Container IP addresses are primarily used for troubleshooting within the local Docker network.

It's important to note that:

- Under the default `bridge` network, a container's IP address is only valid within the host machine or the same Docker network.
- It is not recommended to use container IP addresses as long-term, stable addresses.
- For production access, port mapping, reverse proxies, or orchestration platforms like Kubernetes Services are usually preferred.

---

## Scenario 33: Troubleshooting Conflicts Between Container IP Ranges and VPCs/Intranets

### Symptoms

- Containers cannot access certain intranet resources.
- Abnormal routing behavior.
- Overlapping IP ranges between the cloud-based VPC and Docker's default network.
- Unexpected NAT or forwarding behaviors.

### Common Troubleshooting Commands

To view the Docker network:

```bash
docker network inspect bridge
```

To check the host machine's routing table:

```bash
ip route
```

To view the container's internal routing table:

```bash
docker exec -it containerID ip route
```

### Explanation

This is a common issue in production environments, especially when cloud-based VPCs, self-built IDC IP ranges, and Docker's default network overlap.

For example, if Docker's default network range overlaps with the company's intranet, cloud-based VPC, or VPN IP range, it may cause containers to have access issues to certain addresses.

During troubleshooting, consider the following aspects:

- Docker bridge network range
- Host machine routing table
- Container internal routing table
- VPC/IDC IP range planning
- VPN IP range
- Security group/firewall rules

---

## Scenario 34: Modifying the Default Docker Network Range

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

### Explanation

- `bip`: Specifies the default bridge network interface card address.
- `default-address-pools`: Defines custom network address pools.
- This configuration helps avoid conflicts with existing VPC/intranet IP ranges.

### Application Method

After making changes, you typically need to restart Docker:

```bash
systemctl restart docker
```

### Notes

Modifying Docker's network configuration may affect existing container networks.

Before making changes in a production environment, confirm the following:

- Whether existing containers depend on the default bridge network.
-```bash
docker port 容器ID
``````bash
docker port container_id
```

To view host port listening:

```bash
ss -tunlp
```

---

## Host Network Mode

Using the host network mode:

```bash
docker run -d --network host nginx
```

To view host port listening:

```bash
ss -tunlp
```

---

## Volumes

Create a volume:

```bash
docker volume create my-volume
```

List volumes:

```bash
docker volume ls
```

View volume details:

```bash
docker volume inspect my-volume
```

Delete a volume:

```bash
docker volume rm my-volume
```

Mount a volume:

```bash
docker run -d \
  -v my-volume:/data \
  nginx
```

Bind a host directory:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html \
  nginx
```

View container mount information:

```bash
docker inspect container_id | grep Mounts -A 20
```

View the bound host directory:

```bash
ls -ld /data/nginx
```

Create a host directory:

```bash
mkdir -p /data/nginx
```

---

## Summary

The core concepts of Docker networking, ports, and volumes are as follows:

- **Networking**: Determines how containers communicate with each other.
- **Port Mapping**: Allows external access to container services.
- **Volumes**: Ensure data persistence across container restarts.

Key steps for troubleshooting networking issues:

```text
docker network ls
→ docker network inspect
→ docker inspect (for container networks)
→ ip route
→ Inside-container ip route
→ Check for IP range conflicts, firewalls, security groups
```

For port issues:

```text
docker ps
→ docker port container_id
→ ss -tunlp
→ Verify bound addresses
→ Check firewalls/security groups
→ Confirm service listening within the container
```

For volume issues:

```text
docker volume ls
→ docker volume inspect
→ docker inspect (for Mounts)
→ Check host directory
→ Verify permissions
→ Check container paths
```

Production recommendations:

- Plan Docker IP ranges carefully to avoid conflicts with VPCs, IDCs, VPNs, or K8s networks.
- Control the exposure of ports; don’t expose everything to 0.0.0.0 by default.
- Use the host network mode with caution.
- Use volumes for persistent data storage in production scenarios.
- Bind-mount directories for debugging, configuration, and clear host-path mapping.
- Always confirm that data can be safely discarded before deleting volumes.
```