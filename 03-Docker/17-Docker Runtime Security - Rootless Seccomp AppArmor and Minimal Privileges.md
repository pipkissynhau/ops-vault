# 17-Docker Runtime Security: rootless, seccomp, AppArmor, and Minimal Privileges

#Docker #ContainerSecurity #RuntimeSafe #rootless #seccomp #AppArmor #SELinux #capability #MinimumPermissions #Transport #Clear.

---

## Recommended Path

03-Container Technology/17-Docker Runtime Security: rootless, seccomp, AppArmor, and Minimal Privileges.md

---

## I. Document Explanation

This document organizes content related to Docker runtime security, focusing on:

- What is Docker runtime security
- Why we cannot only focus on image security
- Docker daemon permission risks
- Docker socket risks
- Why the docker group is equivalent to high privileges
- rootless Docker
- user namespace
- seccomp
- AppArmor
- SELinux
- capability minimization
- `--cap-drop ALL`
- `--security-opt no-new-privileges`
- Read-only root filesystem
- `--read-only`
- `--tmpfs`
- Mount restrictions
- Limit privileged
- Limit host network
- Limit Docker socket mounting
- Container runtime security checklist

The goal is:

- To understand Docker runtime security risks
- → Avoid using privileged blindly
- → Understand the value of rootless / user namespace
- → Understand the role of seccomp / AppArmor / SELinux
- → Run containers with minimal capabilities
- → Use read-only root filesystem to reduce risks
- → Establish a Docker production runtime security baseline

---

## II. Why Pay Attention to Docker Runtime Security

The 11th article already organized image security, focusing on:

```text
Basic mirror
Hole scan
SBOM
Trivy
Docker Scout
Harbor Scan
Dockerfile Sensitive information
```

But Docker security is not only about images.

Image security solves:

```text
Are there any holes in the mirror?
Any sensitive files in the mirror?
Whether mirror construction is regulated
```

Runtime security solves:

```text
What are the privileges when the container runs?
Can containers access host sensitive resources?
Can the container break through the quarantine?
Is the container capable of performing a dangerous system call?
Can the container get extra? Linux capability
Can the container write into the unwritten directory?
Can the container be mounted? Docker socket Control host
```

One-sentence understanding:

```text
Mirror clear.
→ Solve“What's the risk in the mirror?”

Runtime Safe
→ Solve“What are the privileges when the container runs?”
```

---

## III. Core Principles of Docker Runtime Security

Docker runtime security can be summarized as:

```text
Minimum permissions
Minimal exposure
Minimum Mount
Minimum capability
Minimum writing
Minimum Trust
```

Production containers should strive to:

```text
Do Not Use privileged
Do Not Mount Docker socket
Do not mount the host root directory
Do Not Use host network
Do Not Use root User Run
Do not grant excess capability
Permission not allowed
Unable to write random root filesystems
Don't expose sensitive catalogues to containers.
```

Security reinforcement directions:

```text
rootless
user namespace
non-root user
cap-drop
seccomp
AppArmor / SELinux
read-only rootfs
no-new-privileges
Resource constraints
Mount Limit
Network Limit
```

---

## IV. Docker Daemon and Docker Socket Risks

---

## Scenario 1: Why Docker Daemon is Sensitive

Docker daemon typically has high privileges.

Common services:

```bash
systemctl status docker
```

Check Docker process:

```bash
ps aux | grep dockerd
```

Understanding:

```text
Docker daemon Can Create Containers
Docker daemon Can Mount Directory
Docker daemon Can operate the network
Docker daemon Can manage mirrors
Docker daemon It controls the container on the host.
```

If an attacker controls Docker daemon, they are usually close to controlling the host.

---

## Scenario 2: Why the docker Group is Dangerous

Many systems add users to the docker group to allow ordinary users to execute Docker commands.

Example:

```bash
usermod -aG docker Username
```

Check user groups:

```bash
id Username
```

Risks:

```text
docker Group users can access Docker socket
Docker socket It can be controlled. Docker daemon
Control Docker daemon It's usually the same as getting high access to the host.
```

Therefore:

```text
Add docker Group ≈ Getting close root Capacity
```

Production recommendation:

```text
Don't just add ordinary users. docker Group
CI/CD Minimize user privileges
Fortress users do not enter by default docker Group
Operations are audited.
```

---

## Scenario 3: What is Docker Socket

Common Docker socket path:

```bash
/var/run/docker.sock
```

Check:

```bash
ls -l /var/run/docker.sock
```

Common output may resemble:

```text
srw-rw---- 1 root docker ...
```

Meaning:

```text
root User and docker Group users can access Docker socket
```

Docker CLI typically communicates with Docker daemon through this socket.

Example:

```bash
docker ps
```

Essentially, it's requesting Docker daemon via Docker socket.

---

## Scenario 4: Risks of Mounting Docker Socket

High-risk writing:

```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  some-image
```

Risks:

```text
The inside program controls the host. Docker daemon
You can create a new high-authorized container.
Can Mount Host Directory
Read host sensitive files
It could cause the container to escape.
```

Extremely high-risk example:

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:latest sh
```

After entering the container, if you can execute:

```bash
docker ps
```

It indicates the container can control the host.

---

## Scenario 5: Why CI/CD Often Mounts Docker Socket

Common needs in CI/CD:

```text
Build mirrors in the current line
In the waterline. docker build
In the waterline. docker push
Start temporary container testing in the current line.
```

So many Jenkins / GitLab Runner mount:

```bash
/var/run/docker.sock
```

But this brings security risks.

Production recommendation:

```text
CI/CD Runner Isolation.
Don't let untrustworthy missions visit. Docker socket
As far as possible, separate projects. runner
Use special build nodes
Limits runner Permissions
Consider rootless buildkit / kaniko / buildx builder Alternatives
```

---

## V. Rootless Docker

---

## Scenario 6: What is Rootless Docker

Rootless Docker refers to:

```text
Docker daemon And the containers are not. root User Run
```

Its goal is to reduce the risk to the host caused by Docker daemon and container runtime vulnerabilities.

Ordinary Docker:

```text
dockerd Here. root Run
Packagings are more capable when running
daemon Controlled high risk
```

Rootless Docker:

```text
dockerd Run as normal user
Container runs in user namespace Internal
Even if there is a problem, try to limit it to normal user privileges. Internal
```

One-sentence understanding:

```text
Rootless Docker = Try not to. root Permissions to run Docker daemon And containers
```

---

## Scenario 7: When is Rootless Docker Suitable

Suitable for:

```text
Develop test environment
Personal Machine
Multi-user shared developers
Building environments with high security requirements
Other Organiser root / docker Group Permissions scene
```

Not necessarily suitable for:

```text
Complex production network scene
The scene of strong dependence on the privileged port
The scenario of strong reliance on kernel capabilities
scenes that require full host access
High performance network demands scenes
old system or cgroup v1 Environment
```

---

## Scenario 8: Common Limitations of Rootless Docker

Common limitations may include:

```text
Network mode behaviour and rootful Docker Not exactly the same.
Low port binding may be restricted
Partial storage drive limited
Part cgroup Resource constraints depend on system support
Some privileged / device It's not the right scene.
There may be differences in performance and network paths.
```

So Rootless is not a universal replacement for all scenarios.

When choosing for production, evaluate:

```text
Operational requirements
Network needs
Performance Requirements
Resource constraints
Security requirements
System Version
cgroup v2 Support
```

---

## Scenario 9: Checking Rootless Docker

Check Docker context:

```bash
docker context ls
```

Check Docker information:

```bash
docker info
```

Check if rootless:

```bash
docker info | grep -i rootless
```

Check current user:

```bash
id
```

Check Docker process:

```bash
ps aux | grep dockerd
```

If dockerd is run by a regular user, it indicates a rootless mode.

---

## VI. user namespace

---

## Scenario 10: What is user namespace

user namespace is used to isolate UID/GID.

It can achieve:

```text
It looks like it's inside. root
It's not real on the host. root
```

Understanding:

```text
Inside the container UID 0
→ Map to host Africa 0 UID
```

This reduces the impact of container root on the host.

---

## Scenario 11: Why user namespace is Valuable

In ordinary containers:

```text
Inside the container root
→ Some scenarios may have a significant impact on the host.
```

After enabling user namespace:

```text
Inside the container root
→ The host is mapped into a normal user range
```

Benefits:

```text
Reducing the risk of escape of containers with extended access
Reduce risk of error in the mount directory
Increasing the security of multi-tenant
```

---

## Scenario 12: Checking User Mapping

Check user inside container:

```bash
docker exec -it ContainersID id
```

Check container main process PID on host:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Check UID mapping:

```bash
cat /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/uid_map
```

Check GID mapping:

```bash
cat /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/gid_map
```

---

## Scenario 13: Basic Understanding of userns-remap /think

Docker supports user namespace remapping.

Configuration file:

```bash
/etc/docker/daemon.json
```

Example:

```json
{
  "userns-remap": "default"
}
```

Restart Docker:

```bash
systemctl restart docker
```

Note:

```text
Enable userns-remap It affects existing containers and mirror data paths.
Assessment and testing of the production environment prior to commissioning
```

Recommendation:

```text
New environmental priority planning
Don't use the old environment directly.
Need to maintain window
Backup required Docker Configure and Data
```

---

## VII. Running Containers as Non-root Users

---

## Scenario 14: Why It's Not Recommended to Run as root Inside Containers

Even without enabling rootless mode, it's recommended to avoid running business containers as root by default.

Risks:

```text
Excessive process privileges in containers
Application gap impact expanded
Mount directory modified by error
Cooperation capability / Mount risk higher
```

---

## Scenario 15: Setting Non-root Users in Dockerfile

Example:

```dockerfile
FROM alpine:3.20

RUN adduser -D appuser

WORKDIR /app

COPY app.sh /app/app.sh

RUN chmod +x /app/app.sh \
    && chown -R appuser:appuser /app

USER appuser

CMD ["/app/app.sh"]
```

Build:

```bash
docker build -t myapp:v1 .
```

Check user after running:

```bash
docker run --rm myapp:v1 id
```

---

## Scenario 16: Specifying User in docker run

Specify UID/GID when running:

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.20 id
```

Explanation:

```text
--user 10001:10001
→ Container process to specify UID/GID Run
```

Note:

```text
Specify Not root After user, the inside directory privileges must match
Otherwise, it's possible. Permission denied
```

---

## Scenario 17: Troubleshooting Permission Issues Caused by Non-root

Symptoms:

```text
Permission denied
```

Troubleshooting:

```bash
docker run --rm myapp:v1 id
```

```bash
docker run --rm myapp:v1 ls -lah /app
```

Enter container:

```bash
docker run --rm -it myapp:v1 /bin/sh
```

Check directory permissions:

```bash
ls -lah /app
```

Resolution:

```dockerfile
RUN chown -R appuser:appuser /app
```

Or ensure host directory permissions match when mounting directories at runtime.

---

## VIII. Capability Minimal Privileges

---

## Scenario 18: Capability Review

Linux capability splits traditional root privileges into finer-grained capabilities.

Common Docker parameters:

```bash
--cap-add
```

```bash
--cap-drop
```

Privileged mode:

```bash
--privileged
```

Differences:

```text
--privileged
→ It's almost all.

--cap-add
→ Increased capacity as required

--cap-drop
→ Remove capacity on demand
```

---

## Scenario 19: Avoid Using Privileged by Default

High risk:

```bash
docker run -d --privileged nginx
```

Risks:

```text
It's too much.
Weakened secure borders
Increased risk of escape from containers
Host device access expanded
Possible operation of the host's inner nuclear capability
```

Production recommendation:

```text
Yes. privileged I don't think so. privileged
Whatever power you need, you add precisely.
```

---

## Scenario 20: Remove All Capabilities and Add as Needed

More strict example:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

Meaning:

```text
Remove All First capability
Add only the capabilities needed to bind low ports
```

Suitable for:

```text
Operating packagings with high safety requirements
Authority requires very clear services
Production safety baseline strengthened
```

---

## Scenario 21: Common Capabilities Added as Needed

Network management capability:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Raw packet capability:

```bash
docker run -d --cap-add NET_RAW nginx
```

Low port binding capability:

```bash
docker run -d --cap-add NET_BIND_SERVICE nginx
```

Process debugging capability:

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

Memory locking capability:

```bash
docker run -d --cap-add IPC_LOCK nginx
```

System management capability:

```bash
docker run -d --cap-add SYS_ADMIN nginx
```

Note:

```text
NET_ADMIN / SYS_ADMIN / SYS_PTRACE You have to be careful.
SYS_ADMIN It's very wide-ranging.
```

---

## Scenario 22: Actively Remove NET_RAW

Ordinary business containers typically don't need raw packet capability.

Can remove:

```bash
docker run -d \
  --cap-drop NET_RAW \
  nginx
```

More strict:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

Benefits:

```text
Decrease original socket Capacity
Lower the net.
Conform to the minimum competence principle
```

---

## IX. no-new-privileges

---

## Scenario 23: What is no-new-privileges

`no-new-privileges` is used to prevent processes from gaining additional privileges through certain mechanisms.

It can be understood as:

```text
Current processes and sub-processes cannot be given more privileges than current ones
```

Docker usage:

```bash
docker run -d \
  --security-opt no-new-privileges:true \
  nginx
```

Effect:

```text
Lower setuid / setgid Waiting for permission to increase risk
Reduce the likelihood that processes in containers will be granted additional privileges
```

---

## Scenario 24: What Containers Suit no-new-privileges

Suitable for:

```text
Normal Web Services
API Services
Static services
Business packagings without privileges
Services with high security baseline requirements
```

Not suitable or should be cautious:

```text
I do. setuid The container of the program.
System tool container that requires special permission to upgrade behavior
```

Production recommendation:

```text
General operating containers are considered for opening by default no-new-privileges
```

Example:

```bash
docker run -d \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

---

## X. seccomp

---

## Scenario 25: What is seccomp

seccomp is a system call filtering mechanism provided by the Linux kernel.

It can be understood as:

```text
seccomp = Which systems can be called by limiting the process
```

System calls are the entry points for applications to execute operations in the kernel.

For example:

```text
mount
ptrace
clone
reboot
keyctl
bpf
```

Some system calls carry higher risks, and if containers don't need them, they can be prohibited via seccomp.

---

## Scenario 26: Docker's Default seccomp Profile

Docker defaults to using a seccomp profile to restrict some system calls for containers.

Check if a container uses the default seccomp profile, typically determined by Docker's default behavior and runtime parameters.

Run a regular container:

```bash
docker run -d nginx
```

By default, the Docker default seccomp profile is applied.

If explicitly disabled:

```bash
docker run -d \
  --security-opt seccomp=unconfined \
  nginx
```

Indicates seccomp restrictions are not used.

Production does not recommend for ordinary business containers:

```bash
--security-opt seccomp=unconfined
```

---

## Scenario 27: Using a Custom seccomp Profile

Assume a seccomp configuration file:

```bash
/seccomp/profile.json
```

Run container:

```bash
docker run -d \
  --security-opt seccomp=/seccomp/profile.json \
  nginx
```

Explanation:

```text
Custom seccomp profile Need to be fully tested
If not, it could result in some system calling being intercepted.
```

---

## Scenario 28: Common Phenomena of seccomp Interception

Common errors:

```text
Operation not permitted
```

Or system call failures in application logs.

Troubleshooting direction:

```text
Whether default is used seccomp
Custom seccomp profile
Is it working? seccomp=unconfined
Whether application requires a special system call
```

Check container configuration:

```bash
docker inspect ContainersID
```

Search for:

```bash
docker inspect ContainersID | grep -i seccomp -A 10
```

---

## Scenario 29: Difference Between seccomp and capability

capability controls:

```text
Competencies of the process
```

seccomp controls:

```text
Could the process call some systems?
```

Differences:

```text
capability
→ Permission dimensions

seccomp
→ System Call Dimensions
```

An operation may be affected by both capability and seccomp.

For example, some debugging or kernel-related operations:

```text
Yes. SYS_PTRACE / SYS_ADMIN Wait. capability
Could be. seccomp profile Limits
```

---

## XI. AppArmor

---

## Scenario 30: What is AppArmor

AppArmor is a Linux security module that uses profiles to restrict which resources programs can access.

It can be understood as:

```text
AppArmor = By procedure profile Binding documents, networks, capabilities, etc.
```

Docker defaults to using:

```text
docker-default
```

This profile provides an additional layer of restriction for containers on systems that support AppArmor.

---

## Scenario 31: Checking AppArmor Status

Check AppArmor service:

```bash
systemctl status apparmor
```

Check AppArmor profile:

```bash
aa-status
```

If there is no `aa-status`, relevant tool packages may need to be installed.

Check Docker's default profile:

```bash
aa-status | grep docker
```

May show:

```text
docker-default
```

---

## Scenario 32: Specifying AppArmor Profile

Specify when running container:

```bash
docker run -d \
  --security-opt apparmor=docker-default \
  nginx
```

If you want to disable AppArmor:

```bash
docker run -d \
  --security-opt apparmor=unconfined \
  nginx
```

Production does not recommend for ordinary business containers:

```bash
--security-opt apparmor=unconfined
```

---

## Scenario 33: Troubleshooting AppArmor Interception

Common phenomena:

```text
Permission denied
Operation not permitted
Failed to apply access to certain paths
Some system operations were rejected
```

Check system logs:

```bash
dmesg | grep -i apparmor
```

Or:

```bash
journalctl -k | grep -i apparmor
```

Troubleshooting direction:

```text
Confirm. AppArmor Enable
Confirm which container is used profile
Confirm whether or not docker-default Limits
Confirm if there is a custom profile
```

---

## Twelve. SELinux

---

## Scenario 34: What is SELinux

SELinux is one of the Mandatory Access Control (MAC) mechanisms in Linux.

It is more commonly found in systems like Red Hat / CentOS / Rocky / AlmaLinux, etc.

It controls which resources processes can access through security contexts and policies.

It can be understood as:

```text
SELinux = Forced access controls based on security tags
```

---

## Scenario 35: Check SELinux Status

Check status:

```bash
getenforce
```

Possible results:

```text
Enforcing
Permissive
Disabled
```

Check detailed status:

```bash
sestatus
```

---

## Scenario 36: SELinux and Docker Mount Issues

On systems with SELinux enabled, permission issues may occur when containers bind mount host directories.

Example:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html \
  nginx
```

The container may not be able to access the mounted directory.

Common solutions are to add label options in the volume parameter.

Example:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:Z \
  nginx
```

Or:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:z \
  nginx
```

Simple understanding:

```text
:Z
→ Sets the mounted directory private SELinux Label

:z
→ Set Share SELinux Label, multiple containers can be shared
```

Testing with system policies is recommended before production use.

---

## Scenario 37: Troubleshooting SELinux Denials

Check kernel logs:

```bash
journalctl -k | grep -i denied
```

Check audit logs:

```bash
ausearch -m avc -ts recent
```

If there is no `ausearch`, the audit tools may need to be installed.

Temporary determination if it's SELinux-related:

```bash
getenforce
```

It is not recommended to disable SELinux directly in production.

Not recommended:

```bash
setenforce 0
```

Unless it's for temporary troubleshooting and the impact scope is clearly known.

---

## Thirteen. Read-Only Root Filesystem

---

## Scenario 38: Why Use a Read-Only Root Filesystem

Regular containers default to having write access to the root filesystem.

Risks:

```text
Applying a loophole may be written into a malicious document
The assailant can modify the contents of the container.
Interim file contaminated container layer
Packagings are growing.
It's hard to separate mirror contents from scripting while running
```

A read-only root filesystem can reduce these risks.

---

## Scenario 39: Using --read-only

Run container:

```bash
docker run -d \
  --read-only \
  nginx
```

Meaning:

```text
Container root file system read-only
```

If the application needs to write to a temporary directory, it may fail.

Example:

```text
/tmp
/var/cache
/var/run
```

It needs to be combined with `--tmpfs` or volume.

---

## Scenario 40: Using tmpfs

Example:

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/cache/nginx \
  nginx
```

Meaning:

```text
Root File System Only
/tmp Use memory temporary file system
/var/cache/nginx Use memory temporary file system
```

Suitable for:

```text
No status service
Static services
Services with high security requirements
```

---

## Scenario 41: Troubleshooting Read-Only Root Filesystem

If the container fails to start, check the logs:

```bash
docker logs ContainersID
```

Common error messages:

```text
Read-only file system
Permission denied
```

Enter the container for testing:

```bash
docker exec -it ContainersID /bin/sh
```

Check the directories needing write access:

```bash
ls -lah /tmp
```

```bash
ls -lah /var/cache
```

Handling:

```text
Identify directory where application really needs to be written
Only mount these directories tmpfs or volume
Keep other root filesystems read-only
```

---

## Fourteen. Limiting Mounts and Sensitive Directories

---

## Scenario 42: High-Risk Mounts

High-risk example:

```bash
docker run -it \
  -v /:/host \
  alpine /bin/sh
```

Risks:

```text
Container access to host root directory
Possible access to sensitive files
Possible changes to host files
It could damage the system.
```

High-risk combination:

```bash
docker run -it \
  --privileged \
  -v /:/host \
  alpine /bin/sh
```

Do not use arbitrarily in production environments.

---

## Scenario 43: Avoid Mounting Sensitive Paths

Try to avoid mounting:

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
 Cloud Factory Business Key Directory
```

If mounting is indeed necessary, it should be done:

```text
Mount only necessary paths
You can read only.
Limit container users
Limits capability
Enable no-new-privileges
Audit.
```

---

## Scenario 44: Read-Only Mounts

Example:

```bash
docker run -d \
  -v /data/config:/app/config:ro \
  myapp:v1
```

Meaning:

```text
/app/config Read only in containers
```

Suitable for:

```text
Profile
Certificate File
Static File
Read-only data
```

---

## Fifteen. Limiting Device Access

---

## Scenario 45: Avoid Brute-Force Privileged Device Access

Not recommended:

```bash
docker run -d --privileged myapp:v1
```

If only a specific device is needed, it's better to use:

```bash
docker run -d \
  --device /dev/sdb:/dev/sdb \
  myapp:v1
```

Explanation:

```text
--device
→ Map only specified devices

--privileged
→ Release a large number of access rights and equipment
```

---

## Scenario 46: Device Access Considerations

Confirm before using device mapping:

```text
Does business really need the equipment?
Appropriate Device Permissions
Access to equipment by container users
Need additional capability
Whether there is a risk of data destruction
```

Check the device:

```bash
ls -lh /dev/sdb
```

Check inside the container:

```bash
docker exec -it ContainersID ls -lh /dev/sdb
```

---

## Sixteen. Limiting Network Exposure

---

## Scenario 47: Avoid Unnecessary Host Network

Not recommended for regular business containers to default use:

```bash
docker run -d --network host nginx
```

Reason:

```text
Network quarantine is down.
Port is directly exposed to host
It's easier to cross the border.
Security borders are weaker.
```

Regular services are better recommended to use:

```bash
docker run -d -p 8080:80 nginx
```

If only local access is needed:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

---

## Scenario 48: Specify Port Binding Range Clearly

Default:

```bash
docker run -d -p 8080:80 nginx
```

Usually equivalent to:

```text
0.0.0.0:8080 -> Containers:80
```

If only local access is desired:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

If only binding to internal IP is needed:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

Production recommendation:

```text
Do not expose management backstage, database, intermediate port brainless 0.0.0.0
```

---

## Seventeen. Resource Limits Are Part of Security

---

## Scenario 49: Why Resource Limits Belong to Security

If a container has no resource limits, abnormal applications may:

```text
Full CPU
Eat all your memory.
Create a large number of processes
Write Blast Disk
Impact on other host services
```

Therefore, resource limits are also part of the runtime security baseline.

---

## Scenario 50: Limit CPU, Memory, and Process Count

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

Limit process count:

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

Check resources:

```bash
docker stats
```

---

## Eighteen. Secure Docker Compose Writing

---

## Scenario 51: Compose Minimum Privilege Example

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "127.0.0.1:8080:80"
    read_only: true
    tmpfs:
      - /tmp
      - /var/cache/nginx
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped
```

Explanation:

```text
read_only
→ Root File System Only

tmpfs
→ Give the necessary temporary directory to write space

cap_drop ALL
→ Remove Default capability

cap_add NET_BIND_SERVICE
→ Only low port binding.

no-new-privileges
→ Prohibition of Additional Permissions

127.0.0.1:8080:80
→ Only host home access is allowed
```

---

## Scenario 52: Avoid High-Risk Compose Configurations

Not recommended:

```yaml
services:
  app:
    image: myapp:v1
    privileged: true
    network_mode: host
    volumes:
      - /:/host
      - /var/run/docker.sock:/var/run/docker.sock
```

Risks:

```text
privileged It's too much.
host network Reduce network isolation
Mount host root is extremely risky
Mount Docker socket Controlled host Docker
```

---

## Nineteen. Kubernetes Correspondence

Although this article focuses on Docker, these concepts also have corresponding items in Kubernetes.

Docker parameter:

```bash
--user
```

Kubernetes:

```yaml
securityContext:
  runAsUser: 10001
```

Docker:

```bash
--cap-drop ALL
```

Kubernetes:

§

## Twenty. Common Issues and Troubleshooting

---

## Issue 1: Operation not permitted in Container

Common causes:

```text
Missing capability
By seccomp Intercept.
By AppArmor Intercept.
By SELinux Intercept.
Not root Insufficient user permissions
no-new-privileges Limits
```

Troubleshooting:

```bash
docker logs ContainersID
```

```bash
docker inspect ContainersID
```

Check kernel logs:

```bash
dmesg | tail -n 50
```

Check AppArmor:

```bash
journalctl -k | grep -i apparmor
```

Check SELinux:

```bash
journalctl -k | grep -i denied
```

---

## Issue 2: Non-root Container Fails to Write Files

Phenomenon:

```text
Permission denied
```

Troubleshooting:

```bash
docker exec -it ContainersID id
```

```bash
docker exec -it ContainersID ls -lah /app
```

Handling:

```dockerfile
RUN chown -R appuser:appuser /app
```

Or adjust the permissions of the host's mounted directory.

---

## Problem 3: read-only causes service startup failure

Phenomenon:

```text
Read-only file system
```

Troubleshooting:

```bash
docker logs ContainersID
```

Identify writeable directories:

```text
/tmp
/var/run
/var/cache
Apply log directory
Apply temporary directory
```

Resolution:

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/cache/nginx \
  nginx
```

---

## Problem 4: seccomp causes system call failure

Troubleshooting:

```bash
docker inspect ContainersID | grep -i seccomp -A 10
```

Temporary verification for seccomp-related issues:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  Mirror Name
```

Note:

```text
seccomp=unconfined Only temporary validation is recommended
Not as a long-term production plan
```

---

## Problem 5: AppArmor causes permission issues

Check AppArmor status:

```bash
aa-status
```

Check kernel logs:

```bash
journalctl -k | grep -i apparmor
```

Temporary verification:

```bash
docker run --rm \
  --security-opt apparmor=unconfined \
  Mirror Name
```

Note:

```text
apparmor=unconfined Only temporary validation is recommended
Production does not recommend long-term closure
```

---

## Problem 6: SELinux causes volume mount unavailability

Check SELinux status:

```bash
getenforce
```

Check denial logs:

```bash
ausearch -m avc -ts recent
```

Attempt to add SELinux label:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:Z \
  nginx
```

Or share label:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:z \
  nginx
```

---

## Problem 7: Mounting Docker socket poses excessive risk

Check container mounts:

```bash
docker inspect ContainersID | grep -A 30 Mounts
```

If you see:

```text
/var/run/docker.sock
```

It indicates the container may control the host Docker.

Recommendation:

```text
Confirm whether or not to mount
Not if you can.
CI/CD runner Segregation
Limitation of available sources of implementation
Use special build nodes
```

---

## Twenty-one, Production Runtime Security Baseline

---

## 1. Recommended Baseline for General Business Containers

Recommended combination:

```bash
docker run -d \
  --user 10001:10001 \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --read-only \
  --tmpfs /tmp \
  --cpus="2" \
  -m 1g \
  --pids-limit 200 \
  -p 127.0.0.1:8080:8080 \
  myapp:v1
```

Explanation:

```text
Not root User
Min capability
Disable Permission Upgrade
Read root-only filesystem
Required Temporary Directory tmpfs
Resource constraints
Process Limit
Port only binds the engine
```

---

## 2. Nginx-like Service Example

```bash
docker run -d \
  --name safe-nginx \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  -p 127.0.0.1:8080:80 \
  nginx:1.27
```

Explanation:

```text
Nginx It may need to be written. /var/cache/nginxI don't know./var/run Waiting Directory
So read root-only file systems together. tmpfs
```

---

## 3. Parameters Not Recommended for Long-term Use

Not recommended for general business containers to use long-term:

```bash
--privileged
```

```bash
--network host
```

```bash
--security-opt seccomp=unconfined
```

```bash
--security-opt apparmor=unconfined
```

```bash
-v /:/host
```

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

```bash
--cap-add SYS_ADMIN
```

```bash
--cap-add ALL
```

---

## 4. Approved Exceptions with Approval Required

Some special components indeed require high privileges, for example:

```text
Monitor collection component
Secure scanning component
Network Component
Storage Component
Backup Component
CI/CD Build Component
```

But must clearly:

```text
Why?
What privileges do you need?
Alternatives available
What's the impact?
Audits available
Is there a quarantine?
Can I just give it to you? cap-add / device
Do you have to? privileged
```

---

## Twenty-two, Common Command Summary

---

## Docker daemon / socket

Check Docker service:

```bash
systemctl status docker
```

Check Docker process:

```bash
ps aux | grep dockerd
```

Check Docker socket:

```bash
ls -l /var/run/docker.sock
```

Check user group:

```bash
id Username
```

Check Docker information:

```bash
docker info
```

---

## Rootless / user namespace

Check context:

```bash
docker context ls
```

Check rootless:

```bash
docker info | grep -i rootless
```

Check container PID:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Check UID mapping:

```bash
cat /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/uid_map
```

Check GID mapping:

```bash
cat /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/gid_map
```

---

## Non-root User

Specify user at runtime:

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.20 id
```

Check container user:

```bash
docker exec -it ContainersID id
```

---

## capability

Remove all capabilities:

```bash
docker run -d \
  --cap-drop ALL \
  nginx
```

Only add low-port binding capability:

```bash
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

Add network management capability:

```bash
docker run -d \
  --cap-add NET_ADMIN \
  nginx
```

Add process debugging capability:

```bash
docker run -d \
  --cap-add SYS_PTRACE \
  nginx
```

---

## no-new-privileges

```bash
docker run -d \
  --security-opt no-new-privileges:true \
  nginx
```

---

## seccomp

Use default seccomp:

```bash
docker run -d nginx
```

Disable seccomp:

```bash
docker run -d \
  --security-opt seccomp=unconfined \
  nginx
```

Specify seccomp profile:

```bash
docker run -d \
  --security-opt seccomp=/seccomp/profile.json \
  nginx
```

---

## AppArmor

Check status:

```bash
aa-status
```

Specify AppArmor:

```bash
docker run -d \
  --security-opt apparmor=docker-default \
  nginx
```

Disable AppArmor:

```bash
docker run -d \
  --security-opt apparmor=unconfined \
  nginx
```

Check logs:

```bash
journalctl -k | grep -i apparmor
```

---

## SELinux

Check status:

```bash
getenforce
```

Check detailed status:

```bash
sestatus
```

Check AVC denial:

```bash
ausearch -m avc -ts recent
```

SELinux label mount:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:Z \
  nginx
```

Shared label mount:

```bash
docker run -d \
  -v /data/nginx:/usr/share/nginx/html:z \
  nginx
```

---

## Read-only Root Filesystem

Read-only run:

```bash
docker run -d \
  --read-only \
  nginx
```

Combined with tmpfs:

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/cache/nginx \
  nginx
```

Read-only mount configuration:

```bash
docker run -d \
  -v /data/config:/app/config:ro \
  myapp:v1
```

---

## Resource Limits

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

Limit process count:

```bash
docker run -d \
  --pids-limit 200 \
  nginx
```

Check resources:

```bash
docker stats
```

---

## Security Check

Check container details:

```bash
docker inspect ContainersID
```

Check mounts:

```bash
docker inspect ContainersID | grep -A 30 Mounts
```

Check security configuration:

```bash
docker inspect ContainersID | grep -i SecurityOpt -A 10
```

Check permission-related logs:

```bash
dmesg | tail -n 50
```

```bash
journalctl -k | grep -i denied
```

---

## Twenty-three, One-sentence Summary

The core of Docker runtime security is:

```text
Don't let the container have more than the business needs.
```

Runtime security mainline:

```text
Not root User
→ Lower Process Permissions

cap-drop ALL
→ Remove Default Capabilities

cap-add Necessary capacity
→ Only capacity that is actually required by the operation

no-new-privileges
→ Disable Permission Upgrade

seccomp
→ Limiting dangerous system calls

AppArmor / SELinux
→ Limit process access resources

read-only rootfs
→ Limit container to write when running

tmpfs / volume
→ Only necessary directory permissions

Resource constraints
→ Preventing misuse of resources

Limit Mount and Network Exposure
→ Reduce host risk
```

High-risk behaviors:

```text
--privileged
--network host
--cap-add ALL
--cap-add SYS_ADMIN
seccomp=unconfined
apparmor=unconfined
-v /:/host
-v /var/run/docker.sock:/var/run/docker.sock
```

Recommended direction for general business containers:

```text
Not root
Min capability
no-new-privileges
Default seccomp
Default AppArmor
Read root-only filesystem
Required Directory tmpfs
Resource constraints
Port Exposure As Required
Hang up. Docker socket
Do not host sensitive directories
```

Production recommendations:

```text
Run safe as Docker Part of baseline
Don't. privileged It's like an all-powerful drug.
Don't. Docker socket Exposure to untrustworthy containers
CI/CD Build nodes in quarantine.
When security restrictions cause problems, you should first locate which level of limitation and then minimize the exception.
Docker Security is not a single-point capability, but a combination of mirror security, operational security, network security, authority governance and audit
```