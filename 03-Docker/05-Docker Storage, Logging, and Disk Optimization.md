# 05-Docker Storage, Logging, and Disk Optimization

#Docker #Storage #Logging #Disk Optimization #data-root #json-file #overlay2 #Ops #Troubleshooting

---

## Recommended Path

03-Container Technology/05-Docker Storage, Logging, and Disk Optimization.md

---

## I. Document Overview

This document outlines common Ops practices related to Docker storage, logging, and disk optimization, focusing on:

- Checking Docker disk usage
- Understanding Docker's default storage directory
- Modifying Docker storage directories
- Migrating Docker data directories
- Troubleshooting large container log files
- Limiting Docker log sizes
- Inspecting Docker storage drivers
- Building optimizations and caches
- Investigating Docker-related disk space issues

The goal is to:

- Understand how to monitor Docker disk usage
- Comprehend the role of `/var/lib/docker`
- Be able to migrate Docker data directories
- Set limits on container log sizes
- Know how to work with Docker storage drivers
- Effectively handle Docker-induced disk space problems

---

## II. Docker Storage and Disk Optimization

---

## Scenario 25: Checking Docker Disk Usage

### Command

```bash
docker system df
```

### Purpose

- Displays the usage of images, containers, and volumes
- Useful for identifying disk growth issues

### Notes

When the host disk space starts to increase unexpectedly, you can use:

```bash
docker system df
```

to check Docker-related resource consumption.

Common areas of interest include:

- Images
- Containers
- Local Volumes
- Build Cache

---

## Scenario 26: Docker's Default Storage Directory

Default directory:

```bash
/var/lib/docker
```

### Explanation

- This directory stores images, container layers, volumes, and network metadata.

In production environments, if the system disk is small and Docker resources continue to grow, it may lead to disk fullness.

Common symptoms include:

- High system disk usage
- Docker startup failures
- Unable to create containers
- Failed image pulls
- Application write errors
- Node abnormalities

---

## Scenario 27: Modifying Docker's Storage Directory (data-root)

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

- Moves the Docker data directory from the system disk to a dedicated data disk, preventing `/var/lib/docker` from filling up the system disk.

### Notes

In production environments with limited system disk space, it's recommended to relocate the Docker data directory to an independent data disk, such as:

```bash
/data/docker
```

or:

```bash
/mnt/docker
```

The specific path should be determined based on your server's disk configuration.

---

## Scenario 28: Migrating Docker Data Directories

### Steps

1. Stop Docker:

```bash
systemctl stop docker
```

2. Copy the data:

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

3. Modify `/etc/docker/daemon.json`:

```json
{
  "data-root": "/data/docker"
}
```

4. Restart Docker:

```bash
systemctl start docker
```

5. Verify the directory change:

```bash
docker info | grep "Docker Root Dir"
```

### Notes

- It's advisable to back up your data before migrating in production.
- Ensure Docker is stopped during the migration process to avoid data inconsistencies.
- Check that the destination directory has sufficient space before making changes.
- After migration, verify functionality before deleting the old directory.

---

## Scenario 29: Large Container Log Files

### Viewing Log Files

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

### Explanation

When Docker uses the `json-file` log driver by default, container logs are stored directly on the host file system. If log sizes are not limited, high-traffic containers can quickly fill up disk space.

Common symptoms include:

- Large `/var/lib/docker` directory size
- Large log files under `/var/lib/docker/containers`
- Insufficient system disk space shown by `df -h`
- Host disk alerts despite containers still running
- Applications generating significant amounts of standard output or errors

---

## Scenario 30: Limiting Docker Log Sizes

### Configuration File

```bash
/etc/docker/daemon.json
```

### Example Configuration

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

### Explanation

- `max-size` limits the size of individual log files to 100MB.
- `max-file` specifies that a maximum of 3 log files will be retained.

### Purpose

- Prevents excessive log growth thatRestart Docker:

```bash
systemctl restart docker
```

---

## 3. Migrate the Docker data directory to the data disk

Stop Docker:

```bash
systemctl stop docker
```

Create the target directory:

```bash
mkdir -p /data/docker
```

Copy the original data:

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

Modify the configuration file:

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

## Six, Production Considerations

---

## 1. It is not recommended to directly delete the Docker data directory

Do not perform operations like:

```bash
rm -rf /var/lib/docker
```

Risks:

- Loss of images
- Loss of containers
- Loss of volume data
- Damage to Docker metadata
- Irreparable business impacts

If cleaning is necessary, first confirm:

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

## 2. Stop Docker before migrating the data directory

Before moving the Docker data directory, stop Docker first:

```bash
systemctl stop docker
```

Reasons:

- To prevent writing during container operation
- To avoid inconsistencies in image layer data
- To prevent incomplete copy of volume data
- To avoid damage to Docker metadata

---

## 3. Configure log limits in advance

If business containers generate a large amount of logs, it is recommended to configure log limits before deployment.

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

Otherwise, after deployment, issues such as:

- Single container logs occupying dozens of GB
- System disk being filled up
- Applications unable to write data
- Node failures
- Increased troubleshooting costs

may occur.

---

## 4. Do not abuse the `--no-cache` option

Build without using cache:

```bash
docker build --no-cache -t myapp:latest .
```

Suitable scenarios:

- When suspecting that cache is causing abnormal build results
- When dependencies have updated but the cache has not been refreshed
- When a complete rebuild is required
- When troubleshooting image build issues

Not suitable for:

- Forcing daily builds without cache
- Using it indiscriminately in CI/CD processes
- Frequent use in environments with limited disk and network resources

---

## Seven, Summary of Common Commands Used in This Chapter

---

## Check Disk and Docker Usage

Check host disk usage:

```bash
df -h
```

Check the default Docker directory usage:

```bash
du -sh /var/lib/docker
```

Check Docker disk usage:

```bash
docker system df
```

Check detailed Docker disk usage:

```bash
docker system df -v
```

---

## Check the Docker Root Dir

```bash
docker info | grep "Docker Root Dir"
```

---

## View Container Log Files

For the default Docker data directory:

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

For the migrated Docker data directory:

```bash
ls -lh /data/docker/containers/*/*.log
```

---

## Find Large Files

For the default Docker data directory:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

For the migrated Docker data directory:

```bash
find /data/docker -type f -size +500M -exec ls -lh {} \;
```

---

## Modify the Docker Data Directory

Stop Docker:

```bash
systemctl stop docker
```

Create the target directory:

```bash
mkdir -p /data/docker
```

Copy the original data:

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

##