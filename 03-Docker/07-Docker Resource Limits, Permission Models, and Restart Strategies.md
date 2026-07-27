# 07-Docker Resource Limits, Permission Models, and Restart Strategies

#Docker #Resource Limits #Restart Strategies #Permission Models #privileged #capability #Operations #Security

---

## Recommended Path

03-Container Technology/07-Docker Resource Limits, Permission Models, and Restart Strategies.md

---

## I. Document Overview

This document outlines common operations related to Docker resource limits, container restart strategies, and permission models, focusing on:

- Limiting CPU usage
- Limiting memory usage
- Simultaneously limiting CPU and memory usage
- Checking container resource consumption
- Docker container restart strategies
- `--restart=no`
- `--restart=on-failure`
- `--restart=always`
- `--restart=unless-stopped`
- Privileged mode `--privileged`
- More granular capability control
- `--cap-add`
- `--cap-drop`
- Common capabilities
- Explanation of common capability options
- Device mapping `--device`
- Recommended approaches for permission control

The goal is:

To be able to control container resource usage

→ To understand how to monitor container resource status

→ To comprehend container restart strategies

→ To distinguish between privileged mode and capability-based restrictions

→ To know the functions of common capabilities

→ To avoid unnecessary use of privileged mode

→ To ensure proper control of container permissions from a security perspective

---

## II. Docker Resource Limits and Performance Optimization

---

## Scenario 36: Limiting CPU Usage

### Command

```bash
docker run -d --cpus="1.5" nginx
```

### Meaning

- Limits the container to using at most 1.5 CPU cores.

### Explanation

`--cpus` is used to restrict the maximum amount of CPU resources a container can use.

For example:

```bash
docker run -d --cpus="1.5" nginx
```

This command specifies that the container should not exceed 1.5 CPU cores.

Common uses:

- Preventing a single container from consuming too much CPU
- Limiting resource usage in testing services
- Ensuring resource isolation on shared hosts
- Avoiding situations where abnormal processes exhaust all host CPU resources

---

## Scenario 37: Limiting Memory Usage

### Command

```bash
docker run -d -m 512m nginx
```

### Meaning

- Limits the maximum available memory for the container to 512MB.

### Explanation

`-m` or `--memory` is used to restrict the maximum amount of memory a container can use.

For example:

```bash
docker run -d -m 512m nginx
```

This command ensures that the container uses no more than 512MB of memory.

If a process within the container exceeds this limit, it may be terminated due to Out Of Memory (OOM) errors.

Common uses:

- Preventing containers from consuming unlimited memory
- Avoiding a single service from affecting the entire host
- Controlling resource consumption in testing environments
- Simulating resource-constrained scenarios

---

## Scenario 38: Simultaneously Limiting CPU and Memory Usage

### Command

```bash
docker run -d --cpus="2" -m 1g nginx
```

### Explanation

- Commonly used in production environments
- Helps prevent a single container from consuming excessive host resources

### Operations Understanding

In production settings, it is generally not advisable to completely disable resource limits for containers.

Without these limits, unexpected issues may occur:

- The CPU may be fully utilized
- Memory may be exhausted
- The host performance could decline
- Other containers might be affected
- Business disruptions could increase

Example of simultaneously limiting CPU and memory:

```bash
docker run -d --cpus="2" -m 1g nginx
```

---

## Scenario 39: Checking Container Resource Usage

### Command

```bash
docker stats
```

### Function

- Provides real-time information on CPU, memory, network, and I/O usage.

### Explanation

`docker stats` is a command for monitoring container resource utilization.

It displays:

- CPU usage percentage
- Memory usage amount
- Memory limits
- Network I/O activities
- Disk I/O operations
- Number of running processes (PIDs)

Use cases:

- Checking if the container is consuming excessive resources
- Identifying if the container is using too much CPU
- Monitoring whether memory is approaching its limit
- Preliminarily identifying resource bottlenecks

---

## III. Docker Container Restart Strategies

---

## Scenario 42: Default No Automatic Restart

### Command

```bash
docker run -d --restart=no nginx
```

### Explanation

- The default setting is `no`.
- The container will not automatically restart after it exits.

### Operations Understanding

`--restart=no` indicates that Docker will not attempt to restart the container once it terminates.

Suitable for:

- Temporary testing containersCan change file ownership.
Can bind to low ports.
Can modify routing settings.
Can adjust iptables rules.
Can send raw packets.
Can load kernel modules.
Can alter the system time.
Can execute mount commands.
……

In Docker, you can add a certain capability using:

```bash
--cap-add
```

You can also remove a capability using:

```bash
--cap-drop
```

Example:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

It can also be written as:

```bash
docker run -d --cap-add CAP_NET_ADMIN nginx
```

Both forms are acceptable. Docker supports both the `CAP_` prefix and the non-`CAP_` prefix.

---

## V. Some Capabilities Reserved by Default in Docker

Docker does not completely lack capabilities by default; instead, it reserves some commonly used ones.

Commonly reserved capabilities include:

| Capability | Explanation | Common Use Cases |
|-----------|--------------|-----------------------------|
| `CHOWN`    | Changes file owner and group. | Used for executing `chown` inside containers. |
| `DAC_OVERRIDE` | Bypasses certain file read/write permission checks. | Needed for root access to files. |
| `FOWNER`     | Bypasses certain file owner checks. | Useful for modifying file attributes when not the current owner. |
| `FSETID`    | Does not clear setuid/setgid bits when modifying files. | Necessary for handling files with special permissions. |
| `KILL`      | Bypasses signal sending permission checks. | Used to send signals to processes. |
| `MKNOD`     | Creates device files. | Useful for tasks like `mknod`. |
| `NET_BIND_SERVICE` | Allows binding to low ports below 1024. | Needed for containers listening on ports like 80/443. |
| `NET_RAW`    | Enables use of RAW/PACKET sockets. | Useful for commands like `ping` and sending raw packets. |
| `SETFCAP`     | Sets file capabilities. | Used with `setcap`. |
| `SETGID`     | Modifies a process's GID. | Useful for switching user groups. |
| `SETPCAP`     | Modifies process capabilities. | Needed for capability management. |
| `SETUID`     | Changes a process's UID. | Used for switching users. |
| `SYS_CHROOT` | Allows use of `chroot`. | Useful for changing the root directory. |

### Operational Understanding

These are common default capabilities in regular Docker containers.

Many basic operations can be performed normally inside containers, such as:

```bash
chown
```

```bash
kill
```

```bash
ping
```

```bash
chroot
```

However, whether these capabilities are actually available depends on factors such as:

- Whether the corresponding commands exist in the image.
- Whether the container is running under root.
- Whether the capability has been explicitly removed.
- Whether the host's security policies restrict them.
- Whether AppArmor/SELinux/seccomp are in place.

---

## VI. Capabilities Not Granted by Default but Can Be Added as Needed

These capabilities are usually not provided to containers by default and need to be added using `--cap-add`.

| Capability | Explanation | Common Use Cases | Risk Level |
|-----------|--------------|-------------------|---------------|
| `NET_ADMIN` | Network management capabilities. | Used for modifying routing, iptables, and network interface settings. | High |
| `NET_BROADCAST` | Broadcasting/multicasting-related capabilities. | Less commonly used. | Medium |
| `DAC_READ_SEARCH` | Bypasses file read permission and directory search restrictions. | Useful for reading restricted files. | High |
| `IPC_LOCK`     | Locks memory. | Used in databases, encryption components, and performance optimization scenarios. | Medium |
| `IPC_OWNER`     | Bypasses System V IPC permission checks. | Useful for IPC-related programs. | Medium |
| `SYS_ADMIN` | A wide range of system management capabilities. | Includes operations like mount, namespace management, and other system-level tasks. | Extremely High |
| `SYS_BOOT`    | Allows restarting the system or loading a new kernel. | Should almost never be granted to regular containers. | Extremely High |
| `SYS_MODULE` | Enables loading and unloading kernel modules. | Useful for drivers and kernel-related tasks. | Extremely High |
| `SYS_NICE`     | Adjusts process priorities. | Useful for performance tuning and real-time tasks. | Medium |
| `SYS_PACCT`    | Processes accounting. | Used in system auditing scenarios. | Medium |
| `SYS_PTRACE` | Allows tracking processes. | Useful for tools like strace and gdb for debugging other processes. | High |
| `SYS_RAWIO`     | Raw I/O port operations. |```bash
docker run -d --cap-add SYS_MODULE nginx
```

### Function

- Loads kernel modules
- Unloads kernel modules

### Note

This is a very dangerous feature.

It should not be added to regular service containers.
---

## 8. IPC_LOCK

### Command

```bash
docker run -d --cap-add IPC_LOCK nginx
```

### Function

- Locks memory
- Prevents memory from being swapped out

### Common Use Cases

- Databases
- Encryption programs
- Certain high-performance components
---

## VIII. Usage of ALL

Docker supports `ALL`.

To add all capabilities:

```bash
docker run -d --cap-add ALL nginx
```

To remove all capabilities:

```bash
docker run -d --cap-drop ALL nginx
```

A safer approach is:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

Explanation:

```text
First, remove all capabilities.
Then, only add the capabilities that are truly necessary.
This approach follows the principle of minimum privilege.
```

### Operations and Maintenance Perspective

It is not recommended to use:

```bash
docker run -d --cap-add ALL nginx
```

Because it grants excessive permissions to the container.

It is more advisable to use:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

This ensures that the container only has the minimum necessary permissions.
---

## IX. Device Mapping

---

## Scenario 49: Device Mapping

### Command

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

### Meaning

- Maps a specific device on the host machine to the container
- This is more precise and less invasive than `--privileged`

### Explanation

`--device` allows you to map a device from the host machine into the container.

For example:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

This command maps `/dev/sdb` on the host machine to be used within the container.

Compared to `--privileged`, this method is more specific:

```text
Only the specified device is mapped, not all permissions are granted.
```

---

## X. Recommended Approach to Permission Control

---

## Scenario 50: Recommended Approach to Permission Control

### Principles

```text
Avoid using `--privileged` if possible.
→ Prefer `cap-add` and `cap-drop`.
→ Consider using `--device` only when necessary.
→ Use `--privileged` as a last resort.
```

### In One Sentence

```text
`--privileged` means granting all permissions.
`cap-add` is used to add specific permissions as needed.
`cap-drop` is used to remove unnecessary permissions.
`--device` is for mapping specific devices.
`--privileged` should be the last option considered.
```

### Production Suggestions

Permission control should follow the principle of minimum privilege:

```text
Do not grant additional permissions by default.
→ Determine which capabilities are required for the business.
→ Prefer `cap-add` to add precise permissions.
→ Remove unnecessary capabilities using `cap-drop`.
→ Use `--device` only when devices are needed.
→ Consider `--privileged` as a last option.
```

---

## XI. Recommendations for Using Capability

---

## 1. Avoid Using `--privileged` Directly

It is not recommended to use:

```bash
docker run -d --privileged nginx
```

Instead, consider:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Or:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

---

## 2. Add Capability Only When Needed

The recommended approach is:

```text
First, identify the error.
→ Determine which capability is missing.
→ Add only that capability.
→ Verify if the issue is resolved.
→ Do not grant too many permissions at once.
```

---

## 3. Be Cautious with High-Risk Capabilities

Be particularly cautious when using the following capabilities:

```text
SYS_ADMIN
SYS_MODULE
SYS_RAWIO
SYS_BOOT
SYS_TIME
SYS_PTRACE
NET_ADMIN
BPF
DAC_READ_SEARCH
```

These capabilities can potentially affect the security boundaries of the host machine and should not be granted casually to regular service containers.

---

## 4. Capabilities Are Not Universal

In some cases, adding capabilities alone is not sufficient; device mapping may also be required.

For example, in FUSE scenarios, you might need both:

```bash
--cap-add SYS_ADMIN
```

And:

```bash
--device /dev/fuse
```

Example:

```bash
docker run --rm -it \
  --cap-add SYS_ADMIN \
  --device /dev/fuse \
  sshfs
```

Although automatic restart is possible, it does not address the underlying issues with the application itself. If a container consistently exits abnormally, you should investigate by using the following commands:

```bash
docker logs containerID
```

```bash
docker inspect containerID
```

Do not rely solely on restart strategies to conceal problems.

---

## 4. Privileged mode should not be used by default

It is not recommended to use `--privileged` without careful consideration:

```bash
docker run -d --privileged nginx
```

Risks include:

- Excessive permissions
- Weaker security boundaries
- Increased risk of container escape
- Higher risk of misoperation on the host machine

It is better to use:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Or:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

---

## 5. Use SYS/Admin with extreme caution

Although `--cap-add SYS ADMIN` is a valid option, it grants very broad permissions.

Before using it, ensure you have considered the following:

- Whether it is truly necessary
- If there are alternative solutions
- Whether lower-level permissions could suffice
- The potential risks to the host machine
- Compliance with security best practices

---

## 6. Always start with the minimum required permissions

It is not advisable to immediately use `--privileged` whenever a permission error occurs. A more reasonable approach is:

```text
Identify the specific error message
→ Determine which capability is missing
→ Temporarily test adding the relevant capability
→ Verify if the issue is resolved
→ Fix the configuration with the minimum necessary permissions
```

---

## Summary of Common Commands

---

## Docker Resource Limits

To limit CPU usage:

```bash
docker run -d --cpus="1.5" nginx
```

To limit memory usage:

```bash
docker run -d -m 512m nginx
```

To limit both CPU and memory:

```bash
docker run -d --cpus="2" -m 1g nginx
```

To check container resource usage:

```bash
docker stats
```

---

## Docker Restart Strategies

By default, containers do not restart automatically:

```bash
docker run -d --restart=no nginx
```

To restart a container upon failure:

```bash
docker run -d --restart=on-failure nginx
```

To limit the number of automatic restarts after failures:

```bash
docker run -d --restart=on-failure:3 nginx
```

To always restart a container:

```bash
docker run -d --restart=always nginx
```

To ensure a container restarts only if it is manually stopped:

```bash
docker run -d --restart=unless-stopped nginx
```

---

## Docker Permission Model

Privileged mode:

```bash
docker run -d --privileged nginx
```

To add a capability:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

To remove a capability:

```bash
docker run -d --cap-drop NET_RAW nginx
```

To add all capabilities:

```bash
docker run -d --cap-add ALL nginx
```

To remove all capabilities:

```bash
docker run -d --cap-drop ALL nginx
```

An example of using minimum permissions:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

For device mapping:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

For FUSE-based scenarios:

```bash
docker run --rm -it \
  --cap-add SYS_ADMIN \
  --device /dev/fuse \
  sshfs
```

---

## Common Capability Examples

To add network management capabilities:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

To add raw packet handling capabilities:

```bash
docker run -d --cap-add NET_RAW nginx
```

To remove raw packet handling capabilities:

```bash
docker run -d --cap-drop NET/raw nginx
```

To add the ability to bind to low ports:

```bash
docker run -d --cap-add NET_BIND_SERVICE nginx
```

To add process debugging capabilities:

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

To add memory locking capabilities:

```bash
docker run -d --cap-add IPC_LOCK nginx
```

To add system administration capabilities:

```bash
docker run -d --cap-add SYS_ADMIN nginx
```

---

## Additional Diagnostic Commands

To list all containers:

```bash
docker ps -a
```

To view container logs:

```bash
docker logs containerID
```

To view the last 100 lines of logs:

```bash
docker logs --tail 100 containerID
```

To inspect a container in detail