# 01-Docker Basic Commands and Container Operation Troubleshooting

#Docker #Container #Ops #Troubleshooting #Logs #Network #DNS

---

## Recommended Path

03-Container Technology/01-Docker Basic Commands and Container Operation Troubleshooting.md

---

## I. Document Description

This document organizes Docker basic commands and container operation troubleshooting capabilities from a production environment perspective, focusing on:

- Docker basic commands
- Viewing basic Docker information
- Viewing containers
- Viewing container details
- Viewing container logs
- Entering a container
- Starting, stopping, and deleting containers
- Basic image management
- Cleaning up Docker resources
- Troubleshooting container startup failures
- Troubleshooting issues with containers unable to access the external network
- Troubleshooting port connectivity issues
- Troubleshooting DNS problems

The goal is:

To be practical

→ To be able to monitor status

→ To be able to troubleshoot issues

→ To be able to identify basic container operation problems

---

## II. Docker Basic Commands

---

## Scenario 1: Viewing Basic Docker Information

View the Docker version:

```bash
docker version
```

View detailed information:

```bash
docker info
```

---

## Scenario 2: Viewing Containers

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
docker inspect containerID
```

---

## Scenario 3: Viewing Container Logs

View logs:

```bash
docker logs containerID
```

View in real time:

```bash
docker logs -f containerID
```

View the last 100 lines:

```bash
docker logs --tail 100 -f containerID
```

---

## Scenario 4: Entering a Container

Enter the container shell:

```bash
docker exec -it containerID /bin/bash
```

If the container does not have bash, use:

```bash
docker exec -it containerID /bin/sh
```

---

## Scenario 5: Starting, Stopping, and Deleting Containers

Start a container:

```bash
docker start containerID
```

Stop a container:

```bash
docker stop containerID
```

Restart a container:

```bash
docker restart containerID
```

Delete a container:

```bash
docker rm containerID
```

Force delete:

```bash
docker rm -f containerID
```

---

## Scenario 6: Image Management

View images:

```bash
docker images
```

Pull an image:

```bash
docker pull nginx
```

Delete an image:

```bash
docker rmi imageID
```

---

## Scenario 7: Resource Cleaning Up

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

## III. Common Issues and Troubleshooting

---

## Scenario 101: Container Startup Failure

### Troubleshooting

View logs:

```bash
docker logs containerID
```

View details:

```bash
docker inspect containerID
```

### Explanation

When a container fails to start, prioritize checking two types of information:

- Container logs
- Container details

`docker logs` is more suitable for viewing errors output by the application itself.

`docker inspect` is more suitable for checking container startup parameters, mounts, network settings, environment variables, exit codes, and other information.

---

## Scenario 102: Containers Unable to Access the External Network

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

When a container cannot access the external network, common issues include:

- Abnormal host machine routing
- Abnormal iptables rules
- Abnormal Docker bridge network configuration
- The host machine itself being unable to access the external network
- Abnormal DNS configuration
- Restrictions by the host machine's firewall or security group

When troubleshooting, do not only check inside the container but also examine the host machine's network settings.

---

## Scenario 103: Port Connectivity Issues

### Troubleshooting

View port mappings:

```bash
docker ps
```

Confirm listening on the port:

```bash
ss -tunlp
```

### Explanation

When a port is not accessible, common issues include:

- The container is not running
- Incorrect `-p` port mapping
- The service inside the container is not listening on the target port
- The host machine's port is not listening
- Interference by the firewall or security group
- The accessed host machine IP is incorrect
- The service is only listening on `127.0.0.