# 04-docker cp and Container File Copy

#Docker #dockercp #Containers #DocumentCopy #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/04-docker cp and Container File Copy.md

---

## I. Document Explanation

This document organizes common uses of `docker cp` in Docker, focusing on:

- Copying files from container to host
- Copying files from host to container
- File copying from stopped containers
- Copying directories
- Path writing precautions
- Use of container names and container IDs
- File permission issues
- Why it's not recommended to use `docker cp` for large-scale data migration

The goal is:

Can extract configuration from container

→ Can extract logs from container

→ Can temporarily place files into container

→ Can understand the applicable boundaries of `docker cp`

---

## II. docker cp Basic Usage

---

## Scenario 20: Copying files from container to host

### Command

```bash
docker cp ContainersID:/Path inside the container /Host Path
```

### Example

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

### Meaning

- Copy files from container to host
- Commonly used for:
  - Retrieving configuration files
  - Retrieving logs
  - Troubleshooting issues

---

## Scenario 21: Copying files from host to container

### Command

```bash
docker cp /Host Path ContainersID:/Container Path
```

### Example

```bash
docker cp /root/nginx.conf nginx:/etc/nginx/nginx.conf
```

### Meaning

- Copy files into container
- Commonly used for:
  - Temporary configuration modification
  - File repair
  - Script injection

---

## Scenario 22: File copying works even if container is not running

### Explanation

```text
docker cp Run without container
```

That is:

```text
Stop state can also copy files
```

### Understanding

As long as the container object still exists, even if the container is not running, you can use `docker cp` to copy files.

This is very useful for troubleshooting failed containers, for example:

- Container exits immediately after startup
- Container cannot enter shell
- Need to extract configuration files
- Need to extract log files
- Need to view a generated file inside container

---

## Scenario 23: Copying directories

### Example

```bash
docker cp nginx:/var/log/nginx /root/nginx-log
```

### Explanation

This command will copy the container's `/var/log/nginx` directory to the host's `/root/nginx-log`.

Common uses include:

- Copying log directories
- Copying configuration directories
- Copying temporary troubleshooting file directories
- Copying application-generated file directories

---

## III. Common Precautions

---

## Scenario 24: Common Precautions

### 1. Paths must be written completely

Error:

```bash
docker cp nginx:nginx.conf /root/
```

Correct:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

### Explanation

Absolute paths are recommended for container paths.

In the error writing:

```bash
nginx:nginx.conf
```

It's not clear enough, leading to potential file or path misinterpretation.

In the correct writing:

```bash
nginx:/etc/nginx/nginx.conf
```

Clearly indicates copying from container's `nginx`'s `/etc/nginx/nginx.conf` to host's `/root/`.

---

### 2. Container names can also be used

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

### Explanation

`docker cp` doesn't necessarily have to use container ID; container names can also be used.

For example, if the container name is `nginx`, you can directly write:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

This is more intuitive than using container ID.

---

### 3. Permission issues

File ownership may change after copying.

Need manual adjustment:

```bash
chown
```

```bash
chmod
```

### Example

Modify file ownership:

```bash
chown root:root /root/nginx.conf
```

Modify file permissions:

```bash
chmod 644 /root/nginx.conf
```

### Explanation

After copying files from container to host, you may encounter:

- File ownership not matching expected user
- File permissions not meeting requirements
- Regular users unable to read
- Service processes unable to read
- Files being overly open with security risks

Therefore, after copying, you need to check permissions according to actual scenarios.

---

### 4. Not suitable for large-scale data migration

Reasons:

- `docker cp` is a temporary operation
- Not suitable for long-term synchronization or large data

Recommendation:

```text
volume / Mount directory More appropriate.
```

### Explanation

`docker cp` is more suitable for temporary copying and not recommended as a long-term data synchronization solution.

Scenarios not recommended for long-term reliance on `docker cp` include:

- Database data migration
- Continuous synchronization of large logs
- Synchronization of application upload directories
- Long-term shared data between container and host
- Regular movement of production data

More suitable approaches include:

- Docker volume
- bind mount
- Object storage
- Backup tools
- Log collection systems
- Dedicated data synchronization tools

---

## IV. Operations and Troubleshooting Understanding

---

## 1. Retrieving configuration files from container

Common scenarios:

- Want to view actual effective configuration in container
- Want to compare default configuration in image
- Want to back up container configuration
- Want to troubleshoot if configuration is overridden

Example:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

After copying, you can view on host:

```bash
cat /root/nginx.conf
```

---

## 2. Retrieving log files from container

Common scenarios:

- Container startup failure
- Application logs not output to standard output
- `docker logs` can't see complete logs
- Logs written in container internal files

Example:

```bash
docker cp nginx:/var/log/nginx /root/nginx-log
```

After copying, view:

```bash
ls -lh /root/nginx-log
```

---

## 3. Temporarily placing configuration files into container

Common scenarios:

- Temporarily replace configuration
- Temporarily place test files
- Temporarily inject troubleshooting scripts
- Quickly validate configuration changes in test environment

Example:

```bash
docker cp /root/nginx.conf nginx:/etc/nginx/nginx.conf
```

Note:

- This method is suitable for temporary verification
- Not recommended as formal release method
- Modifications may be lost after container restart
- Formal environments recommend handling through image, mounting, configuration management, or orchestration systems

---

## 4. Still can retrieve files after container exits

If the container has exited but the container object still exists:

```bash
docker ps -a
```

You can still execute:

```bash
docker cp ContainersID:/Path inside the container /Host Path
```

This is very useful for troubleshooting startup failures.

For example:

```bash
docker cp ContainersID:/app/logs /root/app-logs
```

---

## V. Common Troubleshooting Approaches

---

## 1. First confirm if the container exists

```bash
docker ps -a
```

If the container has been deleted, you can no longer use `docker cp` to retrieve files from it.

---

## 2. Confirm if the container path is correct

You can enter the container to check:

```bash
docker exec -it ContainersID /bin/sh
```

Then check the path:

```bash
ls -lh /etc/nginx/
```

If the container cannot be entered but the container object still exists, you can combine `docker inspect` to view partial information:

```bash
docker inspect ContainersID
```

---

## 3. Confirm if the host target path exists

For example:

```bash
ls -ld /root/
```

If the target directory doesn't exist, you need to create it first:

```bash
mkdir -p /root/nginx-log
```

---

## 4. Check file permissions after copying

```bash
ls -lh /root/nginx.conf
```

Adjust permissions as needed:

```bash
chown root:root /root/nginx.conf
```

```bash
chmod 644 /root/nginx.conf
```

---

## VI. Common Commands Summary in This Document

---

## docker cp

From container to host:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

From host to container:

```bash
docker cp /root/nginx.conf nginx:/etc/nginx/nginx.conf
```

Copy container directory to host:

```bash
docker cp nginx:/var/log/nginx /root/nginx-log
```

View all containers, including stopped ones:

```bash
docker ps -a
```

Enter container to confirm path:

```bash
docker exec -it ContainersID /bin/sh
```

View container details: /think

```bash
docker inspect ContainersID
```

Check the host's target directory:

```bash
ls -ld /root/
```

Create the host's target directory:

```bash
mkdir -p /root/nginx-log
```

Check the copied files:

```bash
ls -lh /root/nginx.conf
```

Change file ownership:

```bash
chown root:root /root/nginx.conf
```

Change file permissions:

```bash
chmod 644 /root/nginx.conf
```

---

## Seven. One-sentence Summary

`docker cp`'s core capabilities are:

Retrieve files from the container

→ Place files into the container

→ Retrieve files even after the container stops

→ Suitable for temporary troubleshooting and configuration checks

But it's not suitable for long-term data synchronization or large-scale data migration.

Recommended understanding:

```text
docker cp
→ Temporary copy, disabling evidence, configuration, log extraction

volume / bind mount
→ Long-term data sustainability, catalogue sharing, production data mounted

Logging System
→ Ongoing log collection, centralized retrieval, uniform analysis
```