# 01-Docker Basic Commands and Container Runtime Troubleshooting

#Docker #Containers #Transport #TheBarrier. #Log #Network #DNS

---

## Recommended Path

03-Container Technology/01-Docker Basic Commands and Container Runtime Troubleshooting.md

---

## Section 1: Document Explanation

This document compiles Docker basic commands and container runtime troubleshooting capabilities from a production environment perspective, with key focuses including:

- Docker Basic Commands
- View Docker Basic Information
- View Containers
- View Container Details
- View Container Logs
- Enter Container
- Container Start/Stop/Deletion
- Image Basic Management
- Docker Resource Cleanup
- Container Startup Failure Troubleshooting
- Container Inability to Access Internet Troubleshooting
- Port Unreachability Troubleshooting
- DNS Issue Troubleshooting

The goal is:

Can Use

→ Can Check Status

→ Can Troubleshoot

→ Can Locate Basic Container Runtime Issues

---

## Section 2: Docker Basic Commands

---

## Scenario 1: View Docker Basic Information

View Docker version:

```bash
docker version
```

View detailed information:

```bash
docker info
```

---

## Scenario 2: View Containers

View running containers:

```bash
docker ps
```

View all containers:

```bash
docker ps -a
```

View container details:

```bash
docker inspect ContainersID
```

---

## Scenario 3: View Container Logs

View logs:

```bash
docker logs ContainersID
```

Real-time view:

```bash
docker logs -f ContainersID
```

View last 100 lines:

```bash
docker logs --tail 100 -f ContainersID
```

---

## Scenario 4: Enter Container

Enter container shell:

```bash
docker exec -it ContainersID /bin/bash
```

If container has no bash, use:

```bash
docker exec -it ContainersID /bin/sh
```

---

## Scenario 5: Container Start/Stop/Deletion

Start container:

```bash
docker start ContainersID
```

Stop container:

```bash
docker stop ContainersID
```

Restart container:

```bash
docker restart ContainersID
```

Delete container:

```bash
docker rm ContainersID
```

Force delete:

```bash
docker rm -f ContainersID
```

---

## Scenario 6: Image Management

View images:

```bash
docker images
```

Pull image:

```bash
docker pull nginx
```

Delete image:

```bash
docker rmi MirrorID
```

---

## Scenario 7: Resource Cleanup

Clean up unused resources:

```bash
docker system prune -a
```

Clean up unused images:

```bash
docker image prune
```

Clean up stopped containers:

```bash
docker container prune
```

Clean up unused volumes:

```bash
docker volume prune
```

---

## Section 3: Common Issues and Troubleshooting

---

## Scenario 101: Container Startup Failure

### Troubleshooting

View logs:

```bash
docker logs ContainersID
```

View details:

```bash
docker inspect ContainersID
```

### Explanation

When container startup fails, prioritize checking two types of information:

- Container logs
- Container details

`docker logs` is more suitable for viewing application-specific error outputs.

`docker inspect` is more suitable for viewing container startup parameters, mounts, network, environment variables, exit codes, etc.

---

## Scenario 102: Container Unable to Access Internet

### Troubleshooting

```bash
iptables -L -n
```

```bash
ip route
```

```bash
docker network inspect bridge
```

### Explanation

When container cannot access internet, common directions include:

- Host routing anomalies
- iptables rule anomalies
- Docker bridge network anomalies
- Host itself unable to access internet
- DNS configuration anomalies
- Host firewall or security group restrictions

Troubleshoot not only inside the container but also check host network.

---

## Scenario 103: Port Unreachable

### Troubleshooting

Check mapping:

```bash
docker ps
```

Confirm listening:

```bash
ss -tunlp
```

### Explanation

When port is unreachable, common directions include:

- Container not running
- `-p` port mapping error
- Container service not listening on target port
- Host port not listening
- Firewall or security group interception
- Incorrect host IP accessed
- Service only listens on `127.0.0.1`

Troubleshoot by first confirming:

```bash
docker ps
```

Then confirm host listening:

```bash
ss -tunlp
```

If host port is already listening, continue checking firewall, security group, and service status.

---

## Scenario 104: DNS Issues

### View Container DNS

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

### Docker Daemon DNS Configuration

```json
{
  "dns": ["8.8.8.8"]
}
```

### Explanation

Common manifestations of container DNS issues include:

- Container unable to resolve domain names
- Can ping IP but not domain
- Application fails to access external domain
- Inconsistent DNS behavior across containers

Troubleshoot by first entering container to check `/etc/resolv.conf`:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

If confirmed as Docker daemon DNS configuration issue, configure DNS uniformly in Docker config file.

Configuration file is usually:

```bash
/etc/docker/daemon.json
```

After modification, restart Docker:

```bash
systemctl restart docker
```

---

## Section 4: Common Commands Summary

---

## Docker Basics

View version:

```bash
docker version
```

View information:

```bash
docker info
```

View running containers:

```bash
docker ps
```

View all containers:

```bash
docker ps -a
```

View logs:

```bash
docker logs -f ContainersID
```

Enter container:

```bash
docker exec -it ContainersID /bin/bash
```

View resources:

```bash
docker stats
```

---

## Basic Troubleshooting

View container logs:

```bash
docker logs ContainersID
```

Real-time view container logs:

```bash
docker logs -f ContainersID
```

View last 100 lines of logs:

```bash
docker logs --tail 100 -f ContainersID
```

View container details:

```bash
docker inspect ContainersID
```

View port mapping:

```bash
docker ps
```

View host listening ports:

```bash
ss -tunlp
```

View host routing:

```bash
ip route
```

View iptables rules:

```bash
iptables -L -n
```

View Docker bridge network:

```bash
docker network inspect bridge
```

View container DNS configuration:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

---

## Section 5: One-Sentence Summary

The core of Docker basic operations is not just knowing how to start containers, but forming a complete troubleshooting chain:

Can view Docker status

→ Can view container status

→ Can view logs

→ Can enter container

→ Can view container details

→ Can judge basic issues like ports, network, DNS, startup failure

Common troubleshooting order:

```text
docker ps
→ docker logs
→ docker inspect
→ ss -tunlp
→ ip route
→ iptables
→ docker network inspect
→ Enter container internal authentication
```