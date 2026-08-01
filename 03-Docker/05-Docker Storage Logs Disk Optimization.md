# 05-Docker Storage, Logs, and Disk Optimization

#Docker #Storage #Log #DiskOptimization #data-root #json-file #overlay2 #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/05-Docker Storage, Logs, and Disk Optimization.md

---

## Section 1: Document Overview

This document organizes common operational capabilities related to Docker storage, logs, and disk optimization, with focus on:

- Checking Docker disk usage
- Docker default storage directory
- Modifying Docker storage directory
- Migrating Docker data directory
- Troubleshooting large container log files
- Limiting Docker log size
- Checking Docker storage driver
- Build optimization and caching
- Troubleshooting Docker disk space issues

The goal is:

Can check Docker disk usage

→ Can understand `/var/lib/docker`'s purpose

→ Can migrate Docker data directory

→ Can limit container log size

→ Can understand Docker storage driver

→ Can resolve Docker-related disk space issues

---

## Section 2: Docker Storage and Disk Optimization

---

## Scenario 25: Check Docker Disk Usage

### Command

```bash
docker system df
```

### Purpose

- View image, container, volume disk usage
- Suitable for troubleshooting disk growth issues

### Notes

When host disk space abnormally increases, first use:

```bash
docker system df
```

to check Docker-related resource usage.

Common focus areas include:

- Images
- Containers
- Local Volumes
- Build Cache

---

## Scenario 26: Docker Default Storage Directory

Default directory:

```bash
/var/lib/docker
```

Notes:

- Images
- Container layers
- Volumes
- Network metadata

Typically all reside in this directory.

### Operational Understanding

`/var/lib/docker` is Docker's core data directory.

If system disk is small and Docker images, containers, logs, volumes continuously grow, it may fill the system disk.

Common manifestations:

- High system disk usage
- Docker startup anomalies
- Unable to create containers
- Unable to pull images
- Application write failures
- Node anomalies

---

## Scenario 27: Modify Docker Storage Directory (data-root)

### Configuration File

```bash
/etc/docker/daemon.json
```

### Example

```json
{
  "data-root": "/data/docker"
}
```

### Purpose

- Move Docker data directory from system disk to data disk
- Avoid `/var/lib/docker` filling up system disk

### Notes

In production environments, if system disk capacity is small, it's recommended to plan Docker data directory on an independent data disk.

For example:

```bash
/data/docker
```

or:

```bash
/mnt/docker
```

Specific paths need to be determined based on server disk planning.

---

## Scenario 28: Migrate Docker Data Directory

### Steps

1. Stop Docker

```bash
systemctl stop docker
```

2. Copy original data

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

3. Modify `/etc/docker/daemon.json`

```json
{
  "data-root": "/data/docker"
}
```

4. Start Docker

```bash
systemctl start docker
```

5. Verify directory effectiveness

```bash
docker info | grep "Docker Root Dir"
```

### Notes

- Backup is recommended before migration in production
- Ensure Docker is stopped during migration to avoid data inconsistency
- Confirm `/data/docker`'s disk space is sufficient before modification
- Do not delete old directories immediately after migration; verify stability first

---

## Scenario 29: Large Container Log Files

### View Log Files

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

### Notes

When Docker uses `json-file` log driver by default, container logs are written directly to the host file.

Without log size limits, containers with high-frequency output may fill the disk.

Common manifestations:

- `/var/lib/docker` occupies a large space
- `/var/lib/docker/containers` has large log files
- `df -h` shows insufficient system disk space
- Containers are running but host disk alerts
- Application outputs large amounts of standard output/error

---

## Scenario 30: Limit Docker Log Size

### Configuration File

```bash
/etc/docker/daemon.json
```

### Example

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

### Meaning

- `max-size`: Single log file maximum 100M
- `max-file`: Maximum 3 files retained

### Purpose

- Prevent container logs from growing infinitely and filling the disk

### Effectiveness

After modifying Docker configuration, typically need to restart Docker:

```bash
systemctl restart docker
```

### Notes

This configuration mainly affects new containers.

For existing containers, it's recommended to recreate them and verify if log limits take effect.

---

## Section 3: Docker Storage Driver and Build Cache

---

## Scenario 40: Check Docker Storage Driver

### Command

```bash
docker info | grep Storage
```

### Recommendation

```text
overlay2
```

### Notes

- `overlay2` is currently a common and recommended storage driver
- Generally preferred in production environments

### Operational Understanding

Docker storage driver affects management of image layers and container layers.

Daily operations don't necessarily require deep modification of storage drivers, but knowing how to check the current driver is essential.

View full information:

```bash
docker info
```

Filter only storage-related info:

```bash
docker info | grep Storage
```

---

## Scenario 41: Build Optimization and Caching

Build image:

```bash
docker build -t myapp:latest .
```

Build without cache:

```bash
docker build --no-cache -t myapp:latest .
```

### Notes

- `--no-cache` should be used only when necessary
- Maintain cache for faster builds normally

### Operational Understanding

Docker uses caching when building images.

Benefits of caching:

- Faster image build speed
- Reduce redundant dependency downloads
- Improve local build efficiency

However, caching also consumes disk space.

When disk space is tight, pay attention to:

```bash
docker system df
```

Build Cache usage.

---

## Section 4: Docker Disk Space Troubleshooting Approach

---

## 1. First Check Host Disk

```bash
df -h
```

Check specific directory usage:

```bash
du -sh /var/lib/docker
```

If Docker data directory has been moved to `/data/docker`, check:

```bash
du -sh /data/docker
```

---

## 2. Check Docker's Own Disk Usage

```bash
docker system df
```

For more detailed info, use:

```bash
docker system df -v
```

Focus areas:

- Excessive images
- Too many stopped containers
- Excessive volumes
- Large build cache
- Large container logs

---

## 3. Check Container Log Size

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

If Docker data directory has been modified, search based on actual directory.

For example:

```bash
ls -lh /data/docker/containers/*/*.log
```

You can also check overall container directory size:

```bash
du -sh /var/lib/docker/containers/*
```

---

## 4. Check Docker Root Dir

```bash
docker info | grep "Docker Root Dir"
```

Used to confirm the actual data directory Docker is using.

---

## 5. View Large Files

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

If the Docker data directory is `/data/docker`:

```bash
find /data/docker -type f -size +500M -exec ls -lh {} \;
```

---

## V. Common Optimization Actions

---

## 1. Clean Unused Resources

Clean unused resources:

```bash
docker system prune -a
```

Clean unused images:

```bash
docker image prune
```

Clean stopped containers:

```bash
docker container prune
```

Clean unused volumes:

```bash
docker volume prune
```

### Note

Before performing cleanup, confirm whether the resources are still in use.

Especially:

```bash
docker volume prune
```

may delete data volumes not referenced by containers.

In production environments, confirm the impact scope before execution.

---

## 2. Limit Container Log Size

Configuration file:

```bash
/etc/docker/daemon.json
```

Example configuration:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

Restart Docker:

```bash
systemctl restart docker
```

---

## 3. Migrate Docker Data Directory to Data Disk

Stop Docker:

```bash
systemctl stop docker
```

Create target directory:

```bash
mkdir -p /data/docker
```

Copy original data:

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

Modify configuration file:

```bash
vi /etc/docker/daemon.json
```

Configuration example:

```json
{
  "data-root": "/data/docker"
}
```

Start Docker:

```bash
systemctl start docker
```

Verify:

```bash
docker info | grep "Docker Root Dir"
```

---

## VI. Production Notes

---

## 1. Do Not Directly Delete Docker Data Directory

Do not directly execute similar operations:

```bash
rm -rf /var/lib/docker
```

Risks:

- Image loss
- Container loss
- Volume data loss
- Docker metadata damage
- Business irrecoverable

If cleanup is mandatory, confirm first:

```bash
docker ps -a
```

```bash
docker images
```

```bash
docker volume ls
```

---

## 2. Stop Docker Before Migration

Before migrating Docker data directory, stop Docker first:

```bash
systemctl stop docker
```

Reasons:

- Avoid container writes during operation
- Avoid image layer inconsistency
- Avoid incomplete volume data copy
- Avoid Docker metadata damage

---

## 3. Log Limits Should Be Configured in Advance

If business container log volume is large, recommend configuring log limits before deployment.

Configuration example:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

Otherwise, after deployment may appear:

- Single container log dozens of GB
- System disk full
- Application unable to write
- Node anomalies
- Higher troubleshooting cost

---

## 4. `--no-cache` Should Not Be Overused

Avoid using cache builds:

```bash
docker build --no-cache -t myapp:latest .
```

Suitable scenarios:

- Suspect cache causes build result anomalies
- Dependency updates but cache not refreshed
- Need complete rebuild
- Troubleshoot image build issues

Unsuitable scenarios:

- Force no-cache for daily builds
- CI/CD indiscriminate usage
- Frequent use in disk and network resource constrained environments

---

## VII. Common Commands Summary in This Article

---

## View Disk and Docker Usage

View host disk:

```bash
df -h
```

View default Docker directory usage:

```bash
du -sh /var/lib/docker
```

View Docker disk usage:

```bash
docker system df
```

View detailed Docker disk usage:

```bash
docker system df -v
```

---

## View Docker Root Dir

```bash
docker info | grep "Docker Root Dir"
```

---

## View Container Log Files

Default Docker data directory:

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

Migrated Docker data directory example:

```bash
ls -lh /data/docker/containers/*/*.log
```

---

## Find Large Files

Default Docker data directory:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

Migrated Docker data directory example:

```bash
find /data/docker -type f -size +500M -exec ls -lh {} \;
```

---

## Modify Docker Data Directory

Stop Docker:

```bash
systemctl stop docker
```

Create target directory:

```bash
mkdir -p /data/docker
```

Copy original data:

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

Edit Docker configuration:

```bash
vi /etc/docker/daemon.json
```

Configuration example:

```json
{
  "data-root": "/data/docker"
}
```

Start Docker:

```bash
systemctl start docker
```

Verify:

```bash
docker info | grep "Docker Root Dir"
```

---

## Limit Docker Log Size

Configuration file:

```bash
/etc/docker/daemon.json
```

Configuration example:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

Restart Docker:

```bash
systemctl restart docker
```

---

## View Storage Driver

```bash
docker info | grep Storage
```

View full Docker information:

```bash
docker info
```

---

## Build Image and Cache

Normal build:

```bash
docker build -t myapp:latest .
```

No-cache build:

```bash
docker build --no-cache -t myapp:latest .
```

---

## Clean Resources

Clean unused resources:

```bash
docker system prune -a
```

Clean unused images:

```bash
docker image prune
```

Clean stopped containers:

```bash
docker container prune
```

Clean unused volumes:

```bash
docker volume prune
```

---

## VIII. One-Sentence Summary

The core of Docker storage and disk optimization is:

First know where Docker data is

→ Then see which resources take up space

→ Then determine if it's images, containers, volumes, logs, or build cache

→ Finally choose to clean, migrate, or limit log size

Core path:

```text
df -h
→ docker system df
→ docker info | grep "Docker Root Dir"
→ du -sh /var/lib/docker
→ ls -lh /var/lib/docker/containers/*/*.log
→ Clean up, move or configure log limits based on actual situation
```

Production recommendations:

```text
Do not overload the disk Docker Data
Docker data-root Suggested advance planning
The container log must be limited in size
Clear volume We have to confirm the data.
Migration Docker Data directory must stop before Docker
The official environment is not direct. rm -rf /var/lib/docker
```