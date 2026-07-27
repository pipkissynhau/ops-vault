# 17-Docker Runtime Security: Rootless, seccomp, AppArmor, and Minimum Privileges

#Docker #Container Security #Runtime Security #rootless #seccomp #AppArmor #SELinux #capability #Minimum Privileges #Ops #Security

---

## Recommended Path

03-Container Technology/17-Docker Runtime Security: Rootless, seccomp, AppArmor, and Minimum Privileges.md

---

## I. Document Overview

This document covers Docker runtime security, focusing on the following topics:

- What Docker runtime security entails
- Why image security alone is insufficient
- Risks associated with Docker daemon permissions
- Threats posed by Docker sockets
- How the docker group equates to high privileges
- Rootless Docker
- User namespaces
- seccomp
- AppArmor
- SELinux
- Minimizing capabilities
- Using `--cap-drop ALL`
- Setting `--security-opt no-new-privileges`
- Reading-only root file systems
- Options like `--read-only` and `--tmpfs`
- Limiting mounts
- Restricting privileged operations
- Controlling host networks
- Limiting Docker socket mounts
- A checklist for container runtime security checks

The goal is to:

- Help users understand the risks associated with Docker runtime security
- Prevent unnecessary use of privileged permissions
- Recognize the value of rootless or user namespaces
- Comprehend the role of seccomp, AppArmor, and SELinux
- Learn how to run containers with minimized capabilities
- Utilize reading-only root file systems to reduce risks
- Establish a secure baseline for Docker production environments

---

## II. Why Focus on Docker Runtime Security

In Chapter 11, we discussed image security, which primarily addresses:

```text
Base images
Vulnerability scanning
SBOMs
Trivy
Docker Scout
Harbor scans
Sensitive information in Dockerfiles
```

However, Docker security extends beyond images.

Image security ensures that:

```text
Images are free from vulnerabilities
They don’t contain sensitive files
Their build processes follow best practices
```

Runtime security, on the other hand, addresses questions such as:

```text
What permissions do containers have during runtime?
Can containers access sensitive host resources?
Can they bypass isolation barriers?
Can they execute dangerous system calls?
Can they obtain additional Linux capabilities?
Can they write to unauthorized directories?
Can they control the host using Docker sockets?
```

In summary:

```text
Image security
→ Addresses “what risks exist in images”
Runtime security
→ Determines “what permissions containers have after startup”
```

---

## III. Core Principles of Docker Runtime Security

Docker runtime security can be summarized as follows:

```text
Minimum privileges
Minimal exposure
Limited mounts
Minimized capabilities
Reduced writes
Low trust levels
```

In production environments, containers should strive to:

```text
Avoid using privileged permissions
Do not mount Docker sockets
Refrain from mounting the host’s root directory
Prevent the use of host networks
Run containers without using the root user
Deny unnecessary capabilities
Prevent privilege escalation
Limit writes to the root file system
Keep sensitive directories hidden from containers
```

Security enhancements include:

```text
Rootless execution
User namespaces
Non-root users
Use of `cap-drop`
Application of seccomp and AppArmor/SELinux
Reading-only rootfs
Setting `no-new-privileges`
Resource restrictions
Mount constraints
Network limitations
```

---

## IV. Risks Associated with Docker Daemon and Docker Sockets

---

## Scenario 1: Why the Docker daemon is Vulnerable

The Docker daemon typically operates with high privileges.

Common services related to it include:

```bash
systemctl status docker
```

To view the Docker process, use:

```bash
ps aux | grep dockerd
```

Understanding these points reveals that the Docker daemon has significant control over:

- Container creation
- Directory mounts
- Network operations
- Image management
- Control over containers on the host

If an attacker gains control of the Docker daemon, they can potentially gain control of the entire host.

---

## Scenario 2: The Danger of the docker group

In many systems, ordinary users are added to the docker group to allow them to execute Docker commands. For example:

```bash
usermod -aG docker username
```

To check the user group, use:

```bash
id username
```

The risk here is that users in the docker group can access the Docker socket, which allows them to control the Docker daemon. Controlling the Docker daemon often equates to gaining high-level host privileges.

Therefore, adding ordinary users to the docker group should be avoided in production environments. It’s also important to minimize permissions for CI/CD users and ensure that Bastion Host users are not automatically added to this group. All operations related to Docker should be audited closely.

---

## Scenario 3:If Docker is run by an ordinary user, it indicates a closer approach to the rootless mode.## Scenario 26: Docker's Default seccomp Profile

By default, Docker uses a seccomp profile to restrict certain system calls within containers.

To determine whether a container uses the default seccomp settings, you can typically look at Docker's default behavior and command-line arguments.

Running an ordinary container:

```bash
docker run -d nginx
```

In this case, the default Docker seccomp profile will be applied.

If you explicitly disable it:

```bash
docker run -d \
  --security-opt seccomp=unconfined \
  nginx
```

This means that no seccomp restrictions will be in effect.

However, for production use, it is not recommended to enable `--security-opt seccomp=unconfined` for regular service containers.

---

## Scenario 27: Using a Custom seccomp Profile

Assuming you have a seccomp configuration file:

```bash
/seccomp/profile.json
```

To run a container with this custom profile, use the following command:

```bash
docker run -d \
  --security-opt seccomp=/seccomp/profile.json \
  nginx
```

**Note:** Using a custom seccomp profile requires thorough testing, as it may prevent certain system calls from being executed within your application.

---

## Scenario 28: Common Issues with seccomp Interceptions

Common error messages include:

```text
Operation not permitted
```

Or you might see failure messages related to specific system calls in your application logs.

**Troubleshooting Steps:** 

- Check whether the default seccomp profile is being used.
- Verify if a custom seccomp profile has been defined.
- Ensure that `seccomp=unconfined` is not enabled.
- Consider whether your application requires certain system calls that are restricted by seccomp.

To view a container's configuration, execute:

```bash
docker inspect 容器ID
```

To search for specific settings related to seccomp, use:

```bash
docker inspect 容器ID | grep -i seccomp -A 10
```

---

## Scenario 29: The Difference Between seccomp and Capability

Capability controls:

```text
Which permission capabilities a process has
```

seccomp controls:

```text
Whether a process is allowed to perform certain system calls
```

**Differences:** 

- **capability**: Focuses on permission levels.
- **seccomp**: Targets specific system calls.

An operation may be affected by both capability and seccomp restrictions. For example, some debugging or kernel-related operations may require capabilities like `SYS_PTRACE` or `SYS_ADMIN`, but they might also be restricted by a seccomp profile.

---

## Chapter 11: AppArmor

---

## Scenario 30: What is AppArmor?

AppArmor is a Linux security module that uses profiles to restrict which resources a program can access.

You can think of it as:

```text
AppArmor = Restricts file, network, capability, and other accesses based on program-specific profiles
```

On systems that support AppArmor, Docker uses the default profile `docker-default` by default. This profile provides an additional layer of security restrictions for containers.

---

## Scenario 31: Checking AppArmor Status

To check the AppArmor service status, run:

```bash
systemctl status apparmor
```

To view the currently active AppArmor profiles, use:

```bash
aa-status
```

If `aa-status` is not available, you may need to install the relevant packages.

To check the Docker default profile, execute:

```bash
aa-status | grep docker
```

You might see something like:

```text
docker-default
```

---

## Scenario 32: Specifying an AppArmor Profile

When running a container, you can specify an AppArmor profile using the following command:

```bash
docker run -d \
  --security-opt apparmor=docker-default \
  nginx
```

To disable AppArmor, use:

```bash
docker run -d \
  --security-opt apparmor=unconfined \
  nginx
```

However, for production use, it is not recommended to disable AppArmor for regular service containers.

---

## Scenario 33: Troubleshooting AppArmor Interceptions

Common issues include:

```text
Permission denied
Operation not permitted
Applications fail to access certain paths
Certain system operations are blocked
```

To diagnose these issues, check the system logs:

```bash
dmesg | grep -i apparmor
```

Or:

```bash
journalctl -k | grep -i apparmor
```

**Troubleshooting Steps:** 

- Confirm whether AppArmor is enabled.
- Identify which profile the container is using.
- Check if it is restricted by `docker-default`.
- Verify if there are any custom profiles in use.

---

## Chapter 12: SELinux

---

##-v /:/host \
  alpine /bin/sh

Do not use it casually in a production environment.

---

## Scenario 43: Do Not Mount Sensitive Paths

Try not to mount the following:

```text
/
 /etc
 /root
 /var/run/docker.sock
 /proc
 /sys
 /dev
 /boot
 /var/lib/docker
 ~/.ssh
 kubeconfig
 Cloud vendor key directory
```

If mounting is really necessary, follow these guidelines:

```text
Mount only essential paths.
Use read-only mode if possible.
Limit container user privileges.
Restrict certain capabilities.
Enable the no-new-privileges setting.
Ensure proper auditing.
```

---

## Scenario 44: Read-Only Mounting

Example:

```bash
docker run -d \
  -v /data/config:/app/config:ro \
  myapp:v1
```

Meaning:

```text
/app/config is read-only inside the container.
```

Suitable for:

```text
Configuration files
Certificate files
Static files
Read-only data
```

---

## XV. Limit Device Access

---

## Scenario 45: Avoid Unnecessary Privileged Device Access

It is not recommended to use:

```bash
docker run -d --privileged myapp:v1
```

If you only need a specific device, it is better to use:

```bash
docker run -d \
  --device /dev/sdb:/dev/sdb \
  myapp:v1
```

Explanation:

```text
--device
→ Maps only the specified device.
--privileged
→ Grants extensive permissions and device access.
```

---

## Scenario 46: Precautions for Device Access

Before using device mapping, confirm:

```text
Whether the business truly requires this device.
Whether the device permissions are appropriate.
Whether the container user can access the device.
Whether additional capabilities are needed.
Whether there is a risk of data corruption.
```

View devices:

```bash
ls -lh /dev/sdb
```

View inside the container:

```bash
docker exec -it containerID ls -lh /dev/sdb
```

---

## XVI. Limit Network Exposure

---

## Scenario 47: Avoid Unnecessary Use of the Host Network

It is not recommended that ordinary service containers use it by default:

```bash
docker run -d --network host nginx
```

Reasons:

```text
Reduced network isolation.
Ports are directly exposed on the host machine.
Higher risk of port conflicts.
Weaker security boundaries.
```

For ordinary services, it is better to use:

```bash
docker run -d -p 8080:80 nginx
```

If access is only required from the local machine:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

---

## Scenario 48: Clearly Define the Port Binding Range

Default:

```bash
docker run -d -p 8080:80 nginx
```

This usually means:

```text
0.0.0.0:8080 -> Container:80
```

If access is only allowed from the local machine:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

If it is only intended for a private IP address:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

In production, it is advised not to expose management interfaces, databases, or middleware ports to the public network indiscriminately.

---

## XVII: Resource Limits Are Also Part of Security

---

## Scenario 49: Why Resource Limits Contribute to Security

If containers do not have resource limits, malicious applications may:

```text
Consume all CPU resources.
Eat up all memory.
Create a large number of processes.
Overwrite the disk.
Impact other services on the same host machine.
```

Therefore, resource limits are also an essential part of runtime security measures.

---

## Scenario 50: Limit CPU, Memory, and Number of Processes

Limit CPU:

```bash
docker run -d \
  --cpus="2" \
  nginx
```

Limit memory:

```bash
docker run -d \
  -m 1g \
  nginx
```

Limit the number of processes:

```bash
docker run -d \
  --pids-limit 200 \
  nginx
```

Combined example:

```bash
docker run -d \
  --cpus="2" \
  -m 1g \
  --pids-limit 200 \
  nginx
```

View resources:

```bash
docker stats
```

---

## XVIII: Secure Docker Compose Configuration

---

## Scenario 51--security-opt seccomp=unconfined \
Image Namedocker run -d \
  -v /data/config:/app/config:ro \
  myapp:v1

---

## Resource Limits

Limit CPU usage:

```bash
docker run -d \
  --cpus="2" \
  nginx
```

Limit memory usage:

```bash
docker run -d \
  -m 1g \
  nginx
```

Limit the number of processes:

```bash
docker run -d \
  --pids-limit 200 \
  nginx
```

View resource usage:

```bash
docker stats
```

---

## Security Checks

View container details:

```bash
docker inspect 容器ID
```

Check mounted volumes:

```bash
docker inspect 容器ID | grep -A 30 Mounts
```

Examine security configurations:

```bash
docker inspect 容器ID | grep -i SecurityOpt -A 10
```

View permission-related logs:

```bash
dmesg | tail -n 50
```

```bash
journalctl -k | grep -i denied
```

---

## Summary

The core of Docker runtime security is to ensure that containers do not possess more permissions than necessary for their operations.

Key principles of runtime security include:

- Using non-root users to reduce process privileges.
- Using `cap-drop ALL` to remove default capabilities.
- Adding only essential capabilities using `cap-add`.
- Enforcing `no-new-privileges` to prevent privilege escalation.
- Implementing seccomp to restrict dangerous system calls.
- Utilizing AppArmor or SELinux to control process resource access.
- Using a read-only rootfs to prevent runtime writes.
- Limiting temporary files and volumes to necessary directories only.
- Applying resource limits to prevent abuse.
- Restricting mounted volumes and network exposure to reduce host risks.

High-risk practices should be avoided, such as using `--privileged`, `--network host`, or allowing unrestricted capabilities. For general business containers, it is recommended to use non-root accounts, minimize necessary capabilities, enforce `no-new-privileges`, use default security settings, maintain a read-only rootfs, and apply appropriate resource limits.

In production environments, runtime security should be integrated into Docker best practices. Privileged modes should not be used as a troubleshooting solution, and Docker sockets should not be exposed to untrusted containers. CI/CD nodes should be isolated, and any security restrictions that cause issues should be carefully evaluated to minimize exceptions. Docker security is ultimately about the combined effectiveness of image security, runtime protections, network controls, permission management, and auditing measures.