# 07-Docker Resource Limits, Permission Models, and Restart Policies

#Docker #ResourceConstraints #RestartPolicy #PermissionModel #privileged #capability #Transport #Clear.

---

## Recommended Path

03-Container Technology/07-Docker Resource Limits, Permission Models, and Restart Policies.md

---

## Section 1: Document Overview

This document organizes common operations related to Docker resource limits, container restart policies, and permission models, with key focuses including:

- Limit CPU
- Limit Memory
- Simultaneously limit CPU and Memory
- View container resource usage
- Docker container restart policies
- `--restart=no`
- `--restart=on-failure`
- `--restart=always`
- `--restart=unless-stopped`
- Privileged mode `--privileged`
- More granular capability control
- `--cap-add`
- `--cap-drop`
- Common capabilities
- Capability option explanations
- Device mapping `--device`
- Recommended permission control approach

The goal is:

- To control container resource usage
→ To view container resource status
→ To understand container restart policies
→ To distinguish between privileged and capability
→ To understand common capability functions
→ To avoid blindly using privileged mode
→ To control container permissions from a production security perspective

---

## Section 2: Docker Resource Limits and Performance Optimization

---

## Scenario 36: Limit CPU

### Command

```bash
docker run -d --cpus="1.5" nginx
```

### Meaning

- Limits container to maximum 1.5 CPUs

### Explanation

`--cpus` is used to limit how much CPU resources a container can use.

Example:

```bash
docker run -d --cpus="1.5" nginx
```

Indicates the container can use up to 1.5 CPUs.

Common uses:

- Prevent single container from monopolizing CPU
- Limit test service resource usage
- Resource isolation on shared host
- Avoid abnormal processes filling up host CPU

---

## Scenario 37: Limit Memory

### Command

```bash
docker run -d -m 512m nginx
```

### Meaning

- Limits container's maximum available memory to 512MB

### Explanation

`-m` or `--memory` is used to limit container's maximum available memory.

Example:

```bash
docker run -d -m 512m nginx
```

Indicates the container can use up to 512MB memory.

If container processes exceed memory limits, it may be OOM Killed.

Common uses:

- Prevent container from infinite memory usage
- Avoid single service affecting entire host
- Control test environment resource consumption
- Simulate resource-constrained scenarios

---

## Scenario 38: Simultaneously Limit CPU and Memory

### Command

```bash
docker run -d --cpus="2" -m 1g nginx
```

### Explanation

- Common in production environments
- Prevent single container from monopolizing host resources

### Operations Understanding

In production environments, it's not recommended to have no resource limits for containers.

If containers have no resource limits, abnormal situations may cause:

- CPU filled up
- Memory exhausted
- Host becomes slow
- Other containers affected
- Business failure scope expanded

Example of simultaneous CPU and memory limits:

```bash
docker run -d --cpus="2" -m 1g nginx
```

---

## Scenario 39: View Container Resource Usage

### Command

```bash
docker stats
```

### Function

- Real-time view of CPU, memory, network, and IO usage

### Explanation

`docker stats` is similar to resource monitoring commands at the container level.

Can see:

- CPU usage rate
- Memory usage
- Memory limit
- Network IO
- Disk IO
- PIDs count

Suitable for:

- Checking if container has resource anomalies
- Judging if container uses excessive CPU
- Judging if container memory approaches limit
- Preliminary identification of resource bottlenecks

---

## Section 3: Docker Container Restart Policies

---

## Scenario 42: Default No Auto Restart

### Command

```bash
docker run -d --restart=no nginx
```

### Explanation

- Default is `no`
- Container won't auto restart after exit

### Operations Understanding

`--restart=no` means Docker won't automatically restart the container after it exits.

Suitable scenarios:

- Temporary test containers
- One-time tasks
- Tasks not wanting to auto restart after failure
- Containers with manually controlled lifecycle

---

## Scenario 43: Auto Restart on Failure

### Command

```bash
docker run -d --restart=on-failure nginx
```

Limit retry count:

```bash
docker run -d --restart=on-failure:3 nginx
```

### Explanation

- Only restarts if container exits abnormally
- Suitable for task-oriented containers needing failure retries

### Operations Understanding

`on-failure` automatically restarts only when container exits abnormally.

Examples:

- Process returns non-zero exit code
- Application abnormal crash
- Script execution failure

Limit retry count:

```bash
docker run -d --restart=on-failure:3 nginx
```

Means maximum 3 restarts after failure.

Suitable scenarios:

- Task-oriented containers
- Batch processing tasks
- Services with failure retry needs but no infinite restarts

---

## Scenario 44: Always Auto Restart

### Command

```bash
docker run -d --restart=always nginx
```

### Explanation

- Restart container immediately upon exit
- Docker service restart will also attempt to restart

### Operations Understanding

`always` means Docker will attempt to restart whenever container exits.

Suitable scenarios:

- Long-running services
- Want host restart to automatically recover
- Simple daemon services

Notes:

- May cause repeated restarts if configured incorrectly
- May enter restart loop when application fails
- Needs to combine with logs to identify real causes

Check container status:

```bash
docker ps -a
```

Check container logs:

```bash
docker logs ContainersID
```

---

## Scenario 45: Auto Restart Unless Manually Stopped

### Command

```bash
docker run -d --restart=unless-stopped nginx
```

### Explanation

- Similar to `always`
- But Docker won't auto restart if manually stopped

### Applicable Scenarios

- Long-running services
- Common in production

### Operations Understanding

`unless-stopped` and `always` are similar, but the difference is:

```text
If the container is manual, stop It's...Docker Do not pull up automatically after restarting
```

This strategy is more suitable for long-running services in production that need to preserve manual stop intent.

Common commands:

```bash
docker run -d --restart=unless-stopped nginx
```

---

## Section 4: Docker Permission Models

---

## Scenario 46: Privileged Mode

### Command

```bash
docker run -d --privileged nginx
```

### Meaning

- Gives container almost full host capabilities
- Very high permissions

### Explanation

- Nearly gives container most host capabilities directly
- Not recommended in production environments
- High security risks

### Operations Understanding

`--privileged` can be understood as:

```text
Large-scale liberalization of container access, bringing the container close to host level capability
```

It may allow containers to do many things ordinary containers can't, such as:

- Access more host devices
- Modify some kernel parameters
- Operate low-level system capabilities
- Load or operate certain system-level resources

Risks:

- Increased container escape risks
- Increased host damage risks
- Weakened security boundaries
- High risks in multi-tenant environments
- Difficult to trace permission sources during troubleshooting

Production principles:

```text
Yes. privileged I don't think so. privileged
```

---

## Scenario 47: More Granular Capability Control than Privileged Mode

### Add capability /think

```bash
docker run -d --cap-add NET_ADMIN nginx
```

### Removing Capabilities

```bash
docker run -d --cap-drop NET_RAW nginx
```

### Explanation

- `--cap-add`: Add capabilities as needed
- `--cap-drop`: Remove capabilities as needed
- More granular and safer than `--privileged`

### Operational Understanding

Linux capability can split root permissions into multiple finer-grained capabilities.

Compared to:

```bash
docker run -d --privileged nginx
```

It's recommended to add capabilities as needed:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

This grants smaller permissions with more controlled risks.

---

## Scenario 48: Common Capabilities

### Network Management Capabilities

```bash
--cap-add NET_ADMIN
```

Suitable for:

- Modify routing
- Modify iptables
- Modify network parameters

### Raw Packet Capabilities

```bash
--cap-add NET_RAW
```

Suitable for:

- ping
- Raw socket

### System Management Capabilities

```bash
--cap-add SYS_ADMIN
```

Explanation:

- Larger permissions
- Although lower than `--privileged`, should be used cautiously

### Operational Understanding

Common capability understanding:

```text
NET_ADMIN
→ Network management capability, often required to operate routes,iptablespackagings for network parameters

NET_RAW
→ Original information capability, common in pingOriginal socket Wait for the scene.

SYS_ADMIN
→ A wide range of systems management capabilities, with strong lines of authority, requires caution.
```

`SYS_ADMIN` is not `--privileged`, but its capability scope is large, and it's not recommended to add it arbitrarily.

---

## Scenario 48 - Supplement: Common Docker Capability Options

Linux capability can be understood as:

```text
Put root Disassemble permissions into small ones.
```

Traditional understanding:

```text
root = I can do anything.
```

After introducing capability, it can be split into:

```text
You can change the owner of the document.
Can bind low ports
We can change the route.
I can change it. iptables
Sending original papers.
Can load kernel modules
Can modify system time
Executable mount
……
```

In Docker, you can add a specific capability through:

```bash
--cap-add
```

Or remove a specific capability through:

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

Both formats are supported, and Docker accepts both with and without the `CAP_` prefix.

---

## Five. Docker's Default Retained Capabilities

Docker doesn't start with zero capabilities, but retains some commonly used ones by default.

Common default retained capabilities include:

| capability | Understanding | Common Scenarios |
|---|---|---|
| `CHOWN` | Modify file owner/group | Execute `chown` inside container |
| `DAC_OVERRIDE` | Bypass some file read/write/execute permission checks | Root access to files |
| `FOWNER` | Bypass some file owner checks | Modify attributes of files not owned by current user |
| `FSETID` | Don't clear setuid/setgid bits when modifying files | Special permission files |
| `KILL` | Bypass signal sending permission checks | Send signals to processes |
| `MKNOD` | Create device files | `mknod` |
| `NET_BIND_SERVICE` | Bind to ports below 1024 | Container listens on 80/443 |
| `NET_RAW` | Use RAW/PACKET socket | `ping`, raw packets |
| `SETFCAP` | Set file capabilities | `setcap` |
| `SETGID` | Modify process GID | Switch user group |
| `SETPCAP` | Modify process capabilities | Capability management |
| `SETUID` | Modify process UID | Switch user |
| `SYS_CHROOT` | Use `chroot` | Change root directory |

### Operational Understanding

These capabilities are relatively common default capabilities in ordinary Docker containers.

Therefore, many basic operations can be executed normally in containers, such as:

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

However, actual availability depends on:

- Whether the corresponding command exists in the image
- Whether the container runs as root
- Whether the capability is explicitly dropped
- Whether host security policies restrict
- Whether AppArmor / SELinux / seccomp restrict

| capability | Understanding | Common Scenarios | Risk Level |
|---|---|---|---|
| `NET_ADMIN` | Network Management Capability | Modify routing, modify iptables, modify network interface parameters | High |
| `NET_BROADCAST` | Broadcast, Multicast Related Capabilities | Rarely used | Medium |
| `DAC_READ_SEARCH` | Bypass File Read Permissions and Directory Search Permissions | Read restricted files | High |
| `IPC_LOCK` | Lock Memory | Databases, encryption components, performance scenarios | Medium |
| `IPC_OWNER` | Bypass System V IPC Permission Checks | IPC-related programs | Medium |
| `SYS_ADMIN` | Large System Management Capabilities | mount, namespace, some system-level operations | Very High |
| `SYS_BOOT` | Reboot System, Load New Kernel | Should almost never be granted to regular containers | Very High |
| `SYS_MODULE` | Load, Unload Kernel Modules | Driver, kernel module related | Very High |
| `SYS_NICE` | Adjust Process Priority | Performance tuning, real-time tasks | Medium |
| `SYS_PACCT` | Process Accounting | System audit scenarios | Medium |
| `SYS_PTRACE` | Track Processes | strace, gdb, debug other processes | High |
| `SYS_RAWIO` | Raw I/O Port Operations | Low-level hardware access | Very High |
| `SYS_RESOURCE` | Bypass Resource Limits | Adjust ulimit, resource limits | High |
| `SYS_TIME` | Modify System Time | Time synchronization, special system programs | High |
| `SYS_TTY_CONFIG` | TTY Device Configuration | Terminal device related | Medium |
| `SYSLOG` | Read Kernel Logs and Other Privileged Syslog Operations | System log collection | High |
| `AUDIT_CONTROL` | Control Kernel Auditing | Audit rule management | High |
| `AUDIT_READ` | Read Audit Logs | Security audit | Medium |
| `BLOCK_SUSPEND` | Prevent System Sleep | Special power management scenarios | Medium |
| `BPF` | Execute Privileged BPF Operations | eBPF, observation, security tools | High |
| `PERFMON` | Performance Observation Capability | perf, performance analysis | Medium to High |
| `CHECKPOINT_RESTORE` | Checkpoint / Restore Related Capabilities | CRIU, container migration | Medium to High |
| `LEASE` | Establish Lease on Files | Special file lock scenarios | Medium |
| `LINUX_IMMUTABLE` | Set Immutable File Attributes | `chattr +i` Related Scenarios | High |
| `MAC_ADMIN` | Modify MAC Security Policy | Smack and other LSM scenarios | High |
| `MAC_OVERRIDE` | Bypass MAC Security Restrictions | Smack and other LSM scenarios | High |
| `WAKE_ALARM` | Trigger Alarm to Wake Up System | Special power management scenarios | Medium |

### Notes

These capabilities should not be added arbitrarily.

Especially:

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

These capabilities significantly increase the container's ability to affect the host system.

---

## Seven. Most Common Capabilities in Operations

---

## 1. NET_ADMIN

### Command

```bash
docker run -d --cap-add NET_ADMIN nginx
```

### Purpose

- Modify routing
- Modify iptables
- Modify network interfaces
- Modify network parameters

### Common Scenarios

- Container needs to perform network debugging
- Container needs to operate iptables
- VPN containers
- Network proxy containers
- CNI / network component containers

### Note

```text
NET_ADMIN The scope is greater than the normal operating container.
```

---

## 2. NET_RAW

### Command

```bash
docker run -d --cap-add NET_RAW nginx
```

### Purpose

- Use RAW socket
- Use PACKET socket
- Support some raw packet capabilities

### Common Scenarios

- `ping`
- Packet capture tools
- Raw packet testing

### Note

Some containers cannot ping, which might be related to `NET_RAW` being removed.

You can also reduce the network attack surface of regular business containers by removing `NET_RAW`:

```bash
docker run -d --cap-drop NET_RAW nginx
```

---

## 3. NET_BIND_SERVICE

### Command

```bash
docker run -d --cap-add NET_BIND_SERVICE nginx
```

### Purpose

- Allow binding to ports below 1024

### Common Scenarios

- Container processes directly listen on 80
- Container processes directly listen on 443

### Note

If the container process is not root but needs to listen on 80/443, this capability might be required.

---

## 4. SYS_ADMIN

### Command

```bash
docker run -d --cap-add SYS_ADMIN nginx
```

### Purpose

- Collection of system management capabilities
- Very broad scope
- Includes many sensitive operations

### Common Scenarios

- mount
- Some FUSE file systems
- Special system management tools
- Containers that run like a full system

### Note

```text
SYS_ADMIN It's very extensive. It's considered the closest. privileged Yes. capability One.
```

Do not easily add this to business containers in production:

```bash
--cap-add SYS_ADMIN
```

---

## 5. SYS_PTRACE

### Command

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

### Purpose

- Allow tracing processes
- Support debugging other processes

### Common Scenarios

- `strace`
- `gdb`
- Performance diagnostics
- Process-level debugging

### Note

This capability might leak process information, so it's recommended for temporary troubleshooting in production environments.

For example, temporarily start a container with debugging capabilities to troubleshoot:

```bash
docker run --rm -it --cap-add SYS_PTRACE alpine /bin/sh
```

---

## 6. SYS_TIME

### Command

```bash
docker run -d --cap-add SYS_TIME nginx
```

### Purpose

- Modify system time

### Note

Containers modifying system time might affect the host or other containers, which is a high risk.

Regular business containers should generally not have this capability.

---

## 7. SYS_MODULE

### Command

```bash
docker run -d --cap-add SYS_MODULE nginx
```

### Purpose

- Load kernel modules
- Unload kernel modules

### Note

This is a very dangerous capability.

Regular business containers should not have this added.

---

## 8. IPC_LOCK

### Command

```bash
docker run -d --cap-add IPC_LOCK nginx
```

### Purpose

- Lock memory
- Prevent memory from being swapped

### Common Scenarios

- Databases
- Encryption programs
- Some high-performance components

---

## Eight. Usage of ALL

Docker supports `ALL`.

Adding all capabilities:

```bash
docker run -d --cap-add ALL nginx
```

Removing all capabilities:

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

Meaning:

```text
Remove All First capability
And only the ability that is needed.
```

This approach better aligns with the principle of least privilege.

### Operational Understanding

Not recommended to use:

```bash
docker run -d --cap-add ALL nginx
```

Because it amplifies container privileges.

Preferred: /think

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

This indicates granting the container only the minimum necessary permissions.

---

## Nine. Device Mapping

---

## Scenario 49: Device Mapping

### Command

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

### Meaning

- Maps a specific host device to the container
- More minimal and explicit than `--privileged`

### Explanation

`--device` can map a specific device from the host to the container.

Example:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

Indicates mapping the host's `/dev/sdb` to the container.

Compared to `--privileged`, this approach is more explicit:

```text
Only the specified device, not the full permission.
```

---

## Ten. Recommended Permission Control Approach

---

## Scenario 50: Recommended Permission Control Approach

### Principle

```text
Yes. privileged I don't think so. privileged

→ Priority cap-add / cap-drop

→ Think again. device

→ Last thought. privileged
```

### One-sentence Understanding

```text
privileged = All of it.

cap-add    = On demand

cap-drop   = As required

device     = Specify Device Map
```

### Production Recommendations

Permission control should follow the principle of minimum permissions:

```text
Default does not give extra permissions
→ Clarifying what capacity is required for operations
→ Use Priority cap-add Accurate increase
→ Not needed. cap-drop Remove
→ It's only for equipment. --device
→ Last thought. --privileged
```

---

## Eleven. Capability Usage Recommendations

---

## 1. Do not use privileged to solve permission issues

Not recommended:

```bash
docker run -d --privileged nginx
```

Preferred options:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Or:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

---

## 2. Prioritize adding capabilities as needed

Recommended approach:

```text
Make sure you're wrong.
→ To judge what's lacking.
→ Add Only Correspond capability
→ Verify resolution
→ Don't give too many privileges at once.
```

---

## 3. Be cautious with high-risk capabilities

The following capabilities should be particularly cautious:

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

These capabilities may affect the host's security boundaries and are not suitable for arbitrary business containers.

---

## 4. Capabilities are not a universal solution

Some scenarios require both capabilities and device mapping.

For example, FUSE scenarios may need:

```bash
--cap-add SYS_ADMIN
```

and:

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

---

## 5. Remove unnecessary capabilities

For example, ordinary business containers do not need raw packet capabilities, which can be removed:

```bash
docker run -d --cap-drop NET_RAW nginx
```

A stricter approach:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

This approach is suitable for higher security requirements.

---

## Twelve. Common Troubleshooting Approaches

---

## 1. Troubleshooting High Container Resource Usage

Check container resource usage:

```bash
docker stats
```

Check all containers:

```bash
docker ps -a
```

Check container logs:

```bash
docker logs ContainersID
```

Check container details:

```bash
docker inspect ContainersID
```

Common directions:

- Abnormal application process CPU usage
- Memory leaks
- Excessive log output
- Abnormal request traffic
- No resource limits on the container
- Multiple containers competing for resources on the same host

---

## 2. Troubleshooting Frequent Container Restarts

Check container status:

```bash
docker ps -a
```

Check logs:

```bash
docker logs ContainersID
```

Check recent logs:

```bash
docker logs --tail 100 ContainersID
```

Check container exit code:

```bash
docker inspect ContainersID
```

Common directions:

- Application startup failure
- Configuration file errors
- Port conflicts
- Dependency services unavailable
- Memory limits too small causing OOM
- Startup command errors
- Health check failures
- `--restart` policy causing repeated restarts

---

## 3. Troubleshooting Permission Issues

If operations in the container report permission issues, first confirm whether additional capabilities are truly needed.

Common phenomena:

- Unable to execute `ping`
- Unable to modify routing
- Unable to modify iptables
- Unable to access host devices
- Unable to perform certain system-level operations
- Unable to execute `mount`
- Unable to use `strace`
- Unable to access specific device files

Troubleshooting directions:

```text
Confirm if the business really needs that authority.
→ The judgment is... capability Insufficient or Device Unmapped
→ First try cap-add
→ It's only for equipment. --device
→ Don't go straight up. privileged
```

Example:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Device mapping example:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

---

## 4. Unable to ping or ping command has no permissions

Possible phenomena include:

```text
Operation not permitted
```

Possible causes:

- Container lacks `NET_RAW`
- Host security policy restrictions
- Missing ping command in the image
- Network itself is unreachable

Try:

```bash
docker run --rm -it --cap-add NET_RAW busybox /bin/sh
```

Enter and test:

```bash
ping 8.8.8.8
```

You can also check if capabilities were actively removed:

```bash
docker inspect ContainersID
```

---

## 5. Unable to modify routing or iptables in the container

Possible phenomena include:

```text
Operation not permitted
```

Common causes:

- Missing `NET_ADMIN`

Try:

```bash
docker run --rm -it --cap-add NET_ADMIN alpine /bin/sh
```

Explanation:

```text
NET_ADMIN Larger privileges are recommended only for containers that do require network management capability.
```

---

## 6. Unable to use strace / gdb in the container

Common causes:

- Missing `SYS_PTRACE`
- seccomp policy restrictions
- Container process user permissions insufficient

Try:

```bash
docker run --rm -it --cap-add SYS_PTRACE alpine /bin/sh
```

Explanation:

```text
SYS_PTRACE For temporary debugging, it is not recommended that general operating packagings be given on a permanent basis.
```

---

## Thirteen. Production Notes

---

## 1. Production containers should set resource limits

Do not run containers without resource limits long-term.

At least consider:

```bash
docker run -d --cpus="2" -m 1g nginx
```

Benefits:

- Avoid single container crashing the host
- Control resource boundaries
- Reduce fault impact scope
- Facilitate capacity planning

---

## 2. Too small memory limits can cause issues

Set memory limit:

```bash
docker run -d -m 512m nginx
```

If memory is set too small, may appear:

- Application frequent OOM
- Container repeated restarts
- Request processing failures
- Performance jitter

Therefore, resource limits are not the smaller the better, but should be set according to actual business resource consumption.

---

## 3. Restart policies cannot replace fault troubleshooting

For example:

```bash
docker run -d --restart=always nginx
```

Although it can automatically restart, it cannot solve application issues.

If the container keeps exiting abnormally, should troubleshoot:

```bash
docker logs ContainersID
```

```bash
docker inspect ContainersID
```

Do not rely solely on restart policies to hide problems.

---

## 4. Do not use privileged as a default solution

Not recommended to use blindly:

```bash
docker run -d --privileged nginx
```

Risks:

- Too broad permissions
- Weak security boundaries
- Increased container escape risks
- Increased risk of host being misoperated

More recommended:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Or:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

---

## 5. Be especially cautious with SYS_ADMIN

Although this is a capability:

```bash
--cap-add SYS_ADMIN
```

Its permission scope is very broad.

Before using, should confirm:

- Whether it's truly needed
- Whether there are alternative solutions
- Whether it can use smaller permissions
- Whether it will bring host risks
- Whether it meets security baseline requirements

---

## 6. Start permission troubleshooting from minimal permissions

Not recommended to directly use:

```bash
docker run -d --privileged nginx
```

More reasonable path:

```text
Look at the details.
→ To judge what's missing.
→ Temporary testing cap-add
→ Verify resolution
→ Solidise minimum permission parameters
```

---

## Fourteen. Summary of Common Commands in This Article

---

## Docker Resource Limits

Limit CPU:

```bash
docker run -d --cpus="1.5" nginx
```

Limit memory:

```bash
docker run -d -m 512m nginx
```

Limit both CPU and memory:

```bash
docker run -d --cpus="2" -m 1g nginx
```

Check container resource usage:

```bash
docker stats
```

---

## Docker Restart Policies

Default no automatic restart:

```bash
docker run -d --restart=no nginx
```

Restart automatically on failure:

```bash
docker run -d --restart=on-failure nginx
```

Limit number of failed restarts:

```bash
docker run -d --restart=on-failure:3 nginx
```

Always restart automatically: /think

```bash
docker run -d --restart=always nginx
```

Unless manually stopped, auto-restart:

```bash
docker run -d --restart=unless-stopped nginx
```

---

## Docker Privilege Model

Privileged mode:

```bash
docker run -d --privileged nginx
```

Add capability:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Remove capability:

```bash
docker run -d --cap-drop NET_RAW nginx
```

Add all capabilities:

```bash
docker run -d --cap-add ALL nginx
```

Remove all capabilities:

```bash
docker run -d --cap-drop ALL nginx
```

Minimum privilege example:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

Device mapping:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

FUSE-like scenario example:

```bash
docker run --rm -it \
  --cap-add SYS_ADMIN \
  --device /dev/fuse \
  sshfs
```

---

## Common Capability Examples

Add network management capability:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Add raw packet capability:

```bash
docker run -d --cap-add NET_RAW nginx
```

Remove raw packet capability:

```bash
docker run -d --cap-drop NET_RAW nginx
```

Add low port binding capability:

```bash
docker run -d --cap-add NET_BIND_SERVICE nginx
```

Add process debugging capability:

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

Add lock memory capability:

```bash
docker run -d --cap-add IPC_LOCK nginx
```

Add system management capability:

```bash
docker run -d --cap-add SYS_ADMIN nginx
```

---

## Troubleshooting

View all containers:

```bash
docker ps -a
```

View container logs:

```bash
docker logs ContainersID
```

View last 100 lines of logs:

```bash
docker logs --tail 100 ContainersID
```

View container details:

```bash
docker inspect ContainersID
```

View container resources:

```bash
docker stats
```

---

## Fifteen. One-Sentence Summary

The core of Docker resource limits, restart policies, and privilege model is:

Resources should be limited

→ Restarts should have policies

→ Privileges should be minimized

→ Don't use auto-restart to hide real failures

→ Don't use privileged instead of precise authorization

Resource control recommendations:

```text
CPU Use --cpus Control
Memory Usage -m Control
Run Status Use docker stats View
```

Restart policy recommendations:

```text
Temporary containers
→ --restart=no

Task packaging
→ --restart=on-failure

Long service
→ --restart=unless-stopped

Special scene
→ --restart=always
```

Privilege control recommendations:

```text
Default does not give extra permissions
→ Consider the need for network capacity --cap-add NET_ADMIN
→ Consider it when it takes the original information capacity. --cap-add NET_RAW
→ When listening to a low-end port --cap-add NET_BIND_SERVICE
→ Temporary consideration when debugging process is required --cap-add SYS_PTRACE
→ Use of specific equipment when required --device
→ Yes. FUSE / mount When you're on the scene, be careful. SYS_ADMIN + device
→ Last thought. --privileged
```

Capability understanding:

```text
privileged
→ It's almost all.

cap-add
→ Increased capacity as needed

cap-drop
→ Remove a capability as needed

cap-drop ALL + cap-add xxx
→ Minimum permission model, safer.

device
→ Map only specified host device
```

Production principles:

```text
If you can limit resources, you can limit resources.
It's a precise authorization.
Yes. privileged I don't think so. privileged
Restart strategy is a bottom-up, not a malfunction.
High risk capability Don't give it long-term.
```