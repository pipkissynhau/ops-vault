# 18-Docker Production Governance: Image Lifecycle, Cleanup Strategies, Backup, and Standardized Operations

#Docker #Production Governance #Image Lifecycle #Cleanup Strategies #Backup and Recovery #Standardized Operations #Harbor #Container Governance #Inspection #Operations

---

## Recommended Path

03-Container Technology/18-Docker Production Governance: Image Lifecycle, Cleanup Strategies, Backup, and Standardized Operations.md

---

## I. Document Overview

This document outlines long-term governance practices for Docker production environments, focusing on:

- Why Docker requires production governance
- Scope of Docker production governance
- Image lifecycle management
- Image tag standards
- Image retention policies
- Harbor project management
- Image cleanup strategies
- Container cleanup strategies
-Risks associated with volume cleanup
- Docker log management
-Docker data directory planning
-Docker configuration backup
- Image and offline migration backup
- Volume data backup
- Compose project backup
- Standardized Docker host baselines
-Docker inspection checklists
- Change control procedures
- Fault recovery and governance closed-loop processes

The goal is to:

- Establish a clear framework for Docker production governance
- Standardize image tags and lifecycle management
- Safely clean up images, containers, logs, and caches
- Determine which resources can be cleared and which cannot be discarded arbitrarily
- Back up Docker configurations, images, data volumes, and Compose projects
- Create standardized operational baselines for Docker hosts

---

## II. Why Docker Requires Production Governance

Docker is easy to use:

```bash
docker run -d nginx
```

However, in a production environment, long-term operations often lead to various issues:

```text
An increasing number of images
Old versions not being cleaned up
The `latest` tag being repeatedly overwritten
Harbor projects becoming unorganized
Container logs filling up disks
Containers remaining running after shutdown
Useless volumes left unchecked
Docker data directories consuming system storage
Lack of resource limits for production containers
 Containers running with default root privileges
Abuse of privileged permissions
No backup of Docker configurations
Inability to recover from failures
```

Therefore, Docker production governance is not just about knowing how to use commands; it’s about ensuring:

- Stability during long-term operations
- Controllable resource usage
- Traceability of images
- Data recoverability
- Minimization of permissions
- Rapid fault identification

In short:

```text
Docker production governance = Making the Docker environment maintainable, traceable, cleanable, recoverable, and auditable
```

---

## III. Scope of Docker Production Governance

Docker production governance encompasses:

```text
Image management
Container management
Network management
Storage management
Log management
Security management
Configuration management
Backup and recovery
Inspection and monitoring
Change control
Fault recovery
```

Corresponding areas include:

```text
Image management
→ Tag standards, retention policies, vulnerability scanning, Harbor cleanup
Container management
→ Naming conventions, resource limits, restart strategies, runtime permissions
Storage management
→ Data-root directories, volumes, bind mounts, disk space optimization
Log management
→ JSON file restrictions, log rotation, collection systems
Security management
→ Non-root user access, capability-based security, seccomp, AppArmor, least privilege principles
Configuration management
→ `daemon.json`, `hosts.toml`, Compose files, runtime parameters
Backup and recovery
→ Images, volumes, Compose projects, Docker configurations
Inspection and monitoring
→ Resource usage, disk space, log analysis, abnormal containers, image accumulation
Change control
→ Configuration modifications, Docker restarts, resource cleanup, version upgrades
```

---

## IV. Image Lifecycle Management

---

## Scenario 1: What is the Image Lifecycle?

A production image typically goes through these stages:

```text
Development and building
→ Local testing
→ Security scanning
→ Pushing to Harbor
→ Deployment in the test environment
→ Pre-release verification
→ Production release
→ Rollback retention
→ Historical archiving
→Expiration cleanup
```

Without proper lifecycle management, issues may arise such as:

```text
Not knowing which images are currently in use
Unclear about which images can be deleted
Lack of knowledge about the code version associated with each image
Difficulty in determining which tag to use for rollback
Continuous expansion of Harbor storage
```

---

## Scenario 2: Why Image Tags Are Important

It is not recommended to use only the `latest` tag in production:

```text
The `latest` tag changes frequently
It makes it impossible to accurately track versions
Rollback processes become unclear
It’s difficult to associate images with specific code during troubleshooting
Confusion can occur across different environments
CI/CD processes become unmanageable
```

It is recommended that tags include the following information:

```text
Application name
Environment
Branch name
Commit ID
Pipeline ID
Version number
Build time
```

Examples:

```docker rmi image ID

or:

```bash
docker rmi image name:tag
```

---

## Scenario 13: Clearing Images Based on Time

Clear images that have not been used in 30 days:

```bash
docker image prune -a --filter "until=720h"
```

Explanation:

```text
720h = 30 days
```

Clear images that have not been used in 7 days:

```bash
docker image prune -a --filter "until=168h"
```

Production Recommendation:

```text
First, test the clearing strategy in a testing environment.
Before clearing in the production environment, retain the current version and any rollback versions.
```

---

## Scenario 14: Checking Before Clearing Images

It is recommended to execute the following commands before clearing:

```bash
docker ps -a
```

```bash
docker images
```

```bash
docker system df -v
```

Confirm the following:

```text
Which images are currently being used by running containers?
Are there any images that need to be retained for rollback purposes?
Are there any production version images?
Are there any temporary testing images?
```

---

## VII. Container Cleaning Strategies

---

## Scenario 15: Viewing Stopped Containers

```bash
docker ps -a
```

Filter for stopped containers:

```bash
docker ps -a --filter "status=exited"
```

---

## Scenario 16: Clearing Stopped Containers

Clear all stopped containers:

```bash
docker container prune
```

Force a clean-up:

```bash
docker container prune -f
```

Explanation:

```text
Stopped containers can usually be cleared.
However, make sure to confirm whether it is necessary to retain any relevant data before doing so.
```

---

## Scenario 17: Do Not Haste to Delete Containers During Production Troubleshooting

If a container exits abnormally, do not delete it immediately.

Instead, retain the following information first:

```bash
docker ps -a
```

```bash
docker logs container ID
```

```bash
docker inspect container ID
```

If necessary, copy the logs or configuration files:

```bash
docker cp container ID:/app/logs /tmp/app-logs
```

Only delete the container after confirming that there is no value in retaining it:

```bash
docker rm container ID
```

---

## Scenario 18: Clearing Containers Based on Conditions

Clear containers that have been stopped for 7 days:

```bash
docker container prune --filter "until=168h"
```

Clear a specific container:

```bash
docker rm container ID
```

Force deletion:

```bash
docker rm -f container ID
```

Note:

```text
docker rm -f will directly stop and delete the container.
Make sure to understand the potential impact before using it in production.
```

---

## VIII. Volume Cleaning Strategies and Risks

---

## Scenario 19: Viewing Volumes

```bash
docker volume ls
```

View detailed information:

```bash
docker volume inspect volume name
```

---

## Scenario 20: Why Volumes Cannot Be Deleted Arbitrarily

Volumes may contain the following data:

```text
MySQL data
PostgreSQL data
Redis persistent data
Files uploaded by applications
Business configuration files
Middleware state data
```

High-risk commands:

```bash
docker volume prune
```

```bash
docker compose down -v
```

Risks:

```text
Deleting a volume that is not referenced by any container may result in the loss of database or business data.
```

---

## Scenario 21: Confirming Before Cleaning Volumes

View volumes:

```bash
docker volume ls
```

View detailed volume information:

```bash
docker volume inspect volume name
```

Check which containers are mounted on this volume:

```bash
docker ps -a --filter volume=volume name
```

View container mount details:

```bash
docker inspect container ID | grep -A 30 Mounts
```

---

## Scenario 22: Deleting a Specific Volume

Delete it only after confirming that it is no longer needed:

```bash
docker volume rm volume name
```

If the deletion fails, it may mean that the volume is still being referenced by some containers.

Check for references:

```bash
docker ps -a --filter volume=volume name
```

---

## Scenario 23: Production Principles for Cleaning Volumes

```text
Do not delete volumes whose purpose is unclear.
Do not directly delete volumes related to databases.
Back up volumes before cleaning them.
Confirm the container reference relationships before clearing.
Ensure that the person in charge of the relevant business is aware of the cleanup process.
Prohibit scripts from performing indiscriminate docker volume prune operations.
```

---

## IX. Docker Log Management

---

## Scenario 24: Risls -lah /backup/docker-config-$(date +%F)docker info | grep -i "Logging Driver"View Docker version:

```bash
docker version
```

View Docker information:

```bash
docker info
```

View containers:

```bash
docker ps
```

View all containers:

```bash
docker ps -a
```

---

## Image Management

View images:

```bash
docker images
```

View dangling images:

```bash
docker images -f "dangling=true"
```

View image usage:

```bash
docker system df -v
```

Clean up dangling images:

```bash
docker image prune
```

Clean up unused images from 30 days ago:

```bash
docker image prune -a --filter "until=720h"
```

Delete a specified image:

```bash
docker rmi image_name:tag
```

---

## Container Management

View stopped containers:

```bash
docker ps -a --filter "status=exited"
```

Clean up stopped containers:

```bash
docker container prune
```

Clean up stopped containers from 7 days ago:

```bash
docker container prune --filter "until=168h"
```

Delete a specified container:

```bash
docker rm container_id
```

---

## Volume Management

View volumes:

```bash
docker volume ls
```

View volume details:

```bash
docker volume inspect volume_name
```

View containers that reference a volume:

```bash
docker ps -a --filter volume=volume_name
```

Delete a specified volume:

```bash
docker volume rm volume_name
```

Perform high-risk cleanup:

```bash
docker volume prune
```

---

## Log Management

View container logs:

```bash
docker logs --tail 100 -f container_id
```

View large logs:

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

View Docker log configuration:

```bash
docker info | grep -i "Logging Driver"
```

View daemon.json:

```bash
cat /etc/docker/daemon.json
```

---

## Disk Management

View disks:

```bash
df -h
```

View the Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

View Docker directory usage:

```bash
du -sh /var/lib/docker
```

View Docker resource usage:

```bash
docker system df
```

View detailed usage:

```bash
docker system df -v
```

---

## Configuration Backup

Create a backup directory:

```bash
mkdir -p /backup/docker-config-$(date +%F)
```

Backup Docker configuration:

```bash
cp -a /etc/docker /backup/docker-config-$(date +%F)/
```

Backup containerd configuration:

```bash
cp -a /etc/containerd /backup/docker-config-$(date +%F)/ 2>/dev/null || true
```

Record Docker information:

```bash
docker info > /backup/docker-config-$(date +%F)/docker-info.txt
```

---

## Image Backup

Export an image:

```bash
docker save -o myapp-v1.tar myapp:v1
```

Import an image:

```bash
docker load -i myapp-v1.tar
```

Export multiple images:

```bash
docker save -o app-bundle.tar nginx:1.27 redis:7 mysql:8.0
```

---

## Volume Backup

Backup a volume:

```bash
docker run --rm \
  -v mysql-data:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar czf /backup/mysql-data-$(date +%F).tar.gz -C /data .
```

Restore a volume:

```bash
docker volume create mysql-data-restore
```

```bash
docker run --rm \
  -v mysql-data-restore:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar xzf /backup/mysql-data-2026-04-25.tar.gz -C /data
```

---

## Compose Backup and Restoration

Backup a Compose project:

```bash
tar czf /backup/myapp-compose-$(date +%F).tar.gz -C /opt/compose myapp
```

Restore a Compose project:

```bash
tar xzf /backup/myapp-compose-2026-04-25.tar.gz -C /opt/compose
```

Check the configuration:

```bash
docker compose config
```

Start the service:

```bash
docker compose up -d
```

View the status:

```bash
docker compose ps
```

---

## Clear Cache

Clear builder cache:

```bash
docker builder prune
```

Clear builder cache from 7 days ago:

```bash
docker builder prune --filter "until=168h"
```

Clear build