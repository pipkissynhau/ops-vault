# 15-Docker Underlying Principles: namespace, cgroup, and overlay2

#Docker #ContainerRationale #namespace #cgroup #overlay2 #UnionFS #CopyWhileWriting #MirrorLayer #Transport #BottomPrinciples

---

## Recommended Path

03-Container Technology/15-Docker Underlying Principles: namespace, cgroup, and overlay2.md

---

## I. Document Overview

This document explains three core concepts in Docker's underlying principles:

- namespace
- cgroup
- overlay2

Key points include:

- Why containers are not virtual machines
- Why containers can isolate
- What namespace is responsible for
- What cgroup is responsible for
- What overlay2 is responsible for
- Relationship between container processes and host processes
- Relationship between container filesystem and image layers
- Why images are layered
- Why containers have a writable layer
- What copy-on-write is
- Why volume data remains after container deletion
- Why overlay2 consumes disk space
- Common underlying troubleshooting commands

The goal is:

- To understand Docker container's underlying composition
- → To explain the difference between containers and virtual machines
- → To understand namespace isolation mechanisms
- → To understand cgroup resource limitation mechanisms
- → To understand overlay2 image layering and copy-on-write
- → To correlate Docker command behaviors with Linux underlying mechanisms

---

## II. Containers Are Not Virtual Machines

Containers are not virtual machines.

Virtual machines typically:

```text
Host hardware
→ Host operating system
→ Hypervisor
→ Virtual machine operating system
→ Application process
```

Containers typically:

```text
Host hardware
→ Host Linux Core
→ Docker / containerd
→ Container Process
```

Core differences:

```text
The Virtual Machine has its own complete operating system. Nuclear

Container shared host core
```

Thus containers are lighter:

```text
Start fast.
Small resource costs
The mirror distribution is easy.
Better suited to micro-service and cloud-based deployment
```

But note:

```text
Container isolation is weaker than virtual machines
Container safety depends on host nuclear capabilities
The nuclear hole in the host can affect the container's safe boundary.
```

---

## III. Docker Container's Three Core Components

You can think of Docker containers as the result of combining three types of Linux capabilities:

```text
namespace
→ Isolation.

cgroup
→ Resource constraints and statistics

overlay2
→ Mirror layers and packaging coding layers
```

One-sentence summary:

```text
namespace Let the container“Looks like an independent machine.”

cgroup restricted containers“How much can you use?”

overlay2 Let the container“Have a separate file system view”
```

---

## IV. namespace: Foundation of Container Isolation

---

## Scenario 1: What is namespace

namespace is an isolation mechanism provided by the Linux kernel.

You can think of it as:

```text
namespace = Show the process a quarantined system view
```

For example:

```text
See your process list inside the container
I saw my host inside the container. First Name
I saw my network equipment inside the container.
See your own mount point inside the container
I saw myself inside the container. IPC Resources
```

But fundamentally:

```text
The process in the container is still on the mainframe.
```

These processes are simply placed into different namespaces.

---

## Scenario 2: Common namespace Types

Common namespaces include:

```text
PID namespace
→ Process number isolated.

NET namespace
→ Network stack quarantine

MNT namespace
→ Mount Point Separator

UTS namespace
→ Separate hostname and domain

IPC namespace
→ Segregation of inter-process communication resources

USER namespace
→ Users and groups ID Segregation

CGROUP namespace
→ cgroup View Separator
```

---

## Scenario 3: PID namespace

PID namespace is used to isolate process IDs.

Viewing processes inside a container:

```bash
ps aux
```

May show the container's main process PID as:

```text
1
```

But on the host, it's actually a regular process.

Viewing processes inside the container:

```bash
docker exec -it ContainersID ps aux
```

Host viewing Docker-related processes:

```bash
ps aux | grep Container process keyword
```

Understanding:

```text
Inside the container PID 1
→ From the perspective of the container 1 Process No.

Host PID
→ Real Process in Host Perspective ID
```

---

## Scenario 4: Why does the container exit when the main process exits

Containers typically rely on a foreground main process.

For example:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

If the main process exits, the container will also exit.

Error example:

```dockerfile
CMD ["echo", "hello"]
```

This command finishes execution and the process ends, so the container naturally exits.

Checking container status:

```bash
docker ps -a
```

Checking logs:

```bash
docker logs ContainersID
```

Understanding:

```text
The container is not a virtual machine.
Container life cycle is usually tied to the main process life cycle
```

---

## Scenario 5: NET namespace

NET namespace is used to isolate network stacks.

Each ordinary Docker container typically has its own:

```text
Cybercard
IP Address
Route Table
iptables View
Port listening
DNS Configure
```

Entering the container to view IP:

```bash
docker exec -it ContainersID ip addr
```

Viewing container routing:

```bash
docker exec -it ContainersID ip route
```

Viewing container DNS:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

Understanding:

```text
Inside the container 127.0.0.1
→ The container itself.

Hostage. 127.0.0.1
→ The host himself.

Another container. 127.0.0.1
→ The other container itself.
```

Therefore, containers should not use `127.0.0.1` to communicate with each other.

---

## Scenario 6: Why host network mode is special

Ordinary containers have independent NET namespaces.

Host network mode:

```bash
docker run -d --network host nginx
```

Indicates the container uses the host's network stack.

Understanding:

```text
Normal bridge Mode
→ The container has its own network. namespace

host Mode
→ Container shared host network namespace
```

Therefore, in host mode:

```text
No need. -p Port Map
The container listening port is the host. mouth
Network isolation is declining.
Increased risk of port conflict
```

Checking host listening ports:

```bash
ss -tunlp
```

---

## Scenario 7: MNT namespace

MNT namespace is used to isolate mount points.

The root directory `/` seen inside the container is not the host's actual `/`.

Viewing root directory inside the container:

```bash
docker exec -it ContainersID ls /
```

Host viewing Docker data directory:

```bash
docker info | grep "Docker Root Dir"
```

Default common directories:

```bash
/var/lib/docker
```

Understanding:

```text
The container has its own file system view
But the bottom data is still hosted by the host. Docker Data Directory Management
```

---

## Scenario 8: UTS namespace

UTS namespace is used to isolate hostname and domain name.

Viewing container hostname:

```bash
docker exec -it ContainersID hostname
```

Specifying hostname when running the container:

```bash
docker run -d --hostname app01 nginx
```

Viewing:

```bash
docker exec -it ContainersID hostname
```

Understanding:

```text
The container can have its own. hostname
Do not affect the host hostname
```

---

## Scenario 9: IPC namespace

IPC namespace is used to isolate inter-process communication resources.

Includes:

```text
System V IPC
POSIX message queues
Share Memory
Signal
```

Ordinary business operations rarely directly manipulate IPC namespaces.

But some databases, middleware, and high-performance programs may involve shared memory mechanisms.

---

## Scenario 10: USER namespace

USER namespace is used to isolate user and group IDs.

It can achieve:

```text
It looks like it's inside. root
The host is for normal users or other UID
```

This helps reduce risks of container escape or privilege escalation.

Related capabilities include:

```text
userns-remap
rootless Docker
```

Viewing users inside the container:

```bash
docker exec -it ContainersID id
```

Viewing host process users:

```bash
ps aux | grep Container process keyword
```

Note:

```text
USER namespace Yes. Docker Important basis for operating safety
Follow-up rootless Docker We'll continue to rely on this direction.
```

---

## Scenario 11: Viewing process namespace information

On the host, you can view a process's namespace:

```bash
ls -l /proc/ProcessID/ns
```

Example output may include:

```text
cgroup
ipc
mnt
net
pid
pid_for_children
time
time_for_children
user
uts
```

Viewing current shell's namespace:

```bash
ls -l /proc/$$/ns
```

If you want to view a container process, you need to first find the host's process PID.

Viewing container details:

```bash
docker inspect ContainersID
```

Filtering PID:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Viewing the process's namespace:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/ns
```

---

## V. cgroup: Resource Limitation and Statistics

---

## Scenario 12: What is cgroup

cgroup stands for:

```text
control groups
```

You can think of it as:

```text
cgroup = Linux Resource constraints and statistical mechanisms for the kernel
```

It can limit and statistics:

```text
CPU
Memory
Disk IO
Network-related resources
Number of processes
Device access
```

Docker's resource limitations depend on cgroup.

For example:

```bash
docker run -d --cpus="2" -m 1g nginx
```

Behind the scenes, it's using cgroup to control container processes.

---

## Scenario 13: Docker CPU Limitation and cgroup

Limiting CPU:

```bash
docker run -d --cpus="1.5" nginx
``` /think

Viewing Container Resources:

```bash
docker stats
```

Understanding:

```text
--cpus="1.5"
→ Restraint container maximum 1.5 individual CPU Computation capacity
```

The actual underlying implementation maps to cgroup CPU control parameters.

---

## Scenario 14: Docker Memory Limits and cgroup

Limiting Memory:

```bash
docker run -d -m 512m nginx
```

Viewing Resources:

```bash
docker stats
```

Checking if Container is OOM:

```bash
docker inspect ContainersID | grep -i oom -A 5
```

Understanding:

```text
-m 512m
→ Limit maximum available memory of containers
```

Exceeding the limit may trigger OOM.

---

## Scenario 15: Why Containers Are OOMKilled

When a process inside the container requests memory exceeding the limit, it may be killed by the kernel.

Common Phenomenon:

```text
Container exit
ExitCode Unusual
State.OOMKilled = true
Apply log break
Host dmesg Yes. kill Information
```

Troubleshooting Commands:

```bash
docker ps -a
```

```bash
docker inspect ContainersID | grep -i oom -A 5
```

```bash
dmesg | grep -i kill
```

```bash
journalctl -k | grep -i oom
```

Handling Approach:

```text
Confirm if OOMKilled
→ View container memory limits
→ View applied memory parameters
→ View flow or task peak
→ Adjustment limit or optimized application
```

---

## Scenario 16: pids Limit

Docker can limit the number of processes inside a container.

Example:

```bash
docker run -d --pids-limit 100 nginx
```

Function:

```text
Prevention of abnormalities in containers fork Numerous processes
Lower fork bomb Risk
```

Viewing Container Details:

```bash
docker inspect ContainersID
```

---

## Scenario 17: cgroup v1 and cgroup v2

Linux systems have:

```text
cgroup v1
cgroup v2
```

Simple Understanding:

```text
cgroup v1
→ Old version, multiple. controller Decentralized

cgroup v2
→ New edition, unified tier, more uniform management
```

Checking Current System cgroup:

```bash
stat -fc %T /sys/fs/cgroup
```

Common Results:

```text
cgroup2fs
→ cgroup v2

tmpfs
→ Maybe. cgroup v1
```

Checking Mounts:

```bash
mount | grep cgroup
```

Checking Docker Information:

```bash
docker info | grep -i cgroup
```

May See:

```text
Cgroup Driver
Cgroup Version
```

---

## Scenario 18: systemd and cgroup Driver

Checking Docker cgroup driver:

```bash
docker info | grep -i "Cgroup Driver"
```

Common Values:

```text
systemd
cgroupfs
```

In Kubernetes environments, kubelet and container runtime cgroup drivers must remain consistent, otherwise it may cause node resource management anomalies.

Checking kubelet Configuration:

```bash
ps aux | grep kubelet
```

Or checking kubelet configuration file:

```bash
cat /var/lib/kubelet/config.yaml | grep cgroupDriver
```

---

## Section VI: overlay2: Image Layering and Container File System

---

## Scenario 19: What is overlay2

overlay2 is the commonly used and recommended storage driver for Docker on Linux.

It is responsible for managing:

```text
Mirror Layer
Packaging Writing Layers
Filesystem Merge View
Copy while writing
```

Checking Docker Storage Driver:

```bash
docker info | grep "Storage Driver"
```

Common Output:

```text
Storage Driver: overlay2
```

---

## Scenario 20: Why Images Are Layered

Docker images consist of multiple layers.

For example Dockerfile:

```dockerfile
FROM ubuntu:22.04

RUN apt-get update

RUN apt-get install -y curl

COPY app.sh /app/app.sh
```

Can be understood as:

```text
Basic mirror layer
→ apt update Form a layer.
→ Install curl Form a layer.
→ COPY app.sh Form a layer.
```

Checking Image History:

```bash
docker history Mirror Name
```

Example:

```bash
docker history nginx:1.27
```

---

## Scenario 21: Image Layers Are Read-Only

Image layers are typically read-only layers.

When starting a container, Docker adds a:

```text
Packaging Writing Layers
```

Understanding:

```text
Mirror Layer
→ Read-only, reusable

Container Layer
→ Writeable. Each container is independent.
```

Multiple containers can be started based on the same image.

They share the image read-only layer, but each has its own container writable layer.

---

## Scenario 22: Where Does Container Write File Go

Example:

```bash
docker run -d --name test-nginx nginx:1.27
```

Entering the Container:

```bash
docker exec -it test-nginx /bin/sh
```

Writing a File:

```bash
echo hello > /tmp/test.txt
```

This file will not be written back to the nginx image layer, but instead written to the container's own writable layer.

Deleting the Container:

```bash
docker rm -f test-nginx
```

The container's writable layer is deleted when the container is removed.

Recreating a Container with the Same Name:

```bash
docker run -d --name test-nginx nginx:1.27
```

Checking Again:

```bash
docker exec -it test-nginx ls /tmp/test.txt
```

The file usually does not exist.

---

## Scenario 23: Why Data Disappears After Container Deletion

If data is written to the container's writable layer:

```text
Packaging Delete
→ Packaging Writing Layer Deleting
→ Data lost
```

Therefore, database, uploaded files, and business data should not be stored only in the container's writable layer.

It should use:

```text
volume
bind mount
External storage
Object Storage
Database Backup
```

Example:

```bash
docker run -d \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

---

## Scenario 24: Why volume Data Remains

volume is an independent data volume managed by Docker, not belonging to the container's writable layer.

Checking volume:

```bash
docker volume ls
```

Checking Details:

```bash
docker volume inspect mysql-data
```

Understanding:

```text
Packaging Delete
→ Packaging Writing Layer Deleting

volume
→ Default does not delete with container
```

Unless executed:

```bash
docker volume rm volumeName
```

Or:

```bash
docker compose down -v
```

Therefore, be cautious when deleting volume in production environments.

---

## Scenario 25: What is Copy-on-Write

The common English term for write-time copy:

```text
Copy-on-Write
```

Abbreviation:

```text
CoW
```

Simple Understanding:

```text
Multiple containers share a read-only mirror layer

If the container is just reading files
→ Read mirror layers directly

If the container wants to modify a file
→ Copy the file to the canary layer first.
→ Change a copy in the writeable layer
```

This is write-time copy.

Benefits:

```text
Save Disk Space
Multiple containers shared mirror layers
Start the container faster.
Mirror distribution is more efficient
```

Costs:

```text
There's an extra cost of writing a lot of containers.
Write-intensive data is not suitable for the packaging.
Database data should be used volume
```

---

## Scenario 26: Common Directories of overlay2

Default Docker data directory:

```bash
/var/lib/docker
```

Common directories of overlay2:

```bash
/var/lib/docker/overlay2
```

Checking:

```bash
ls -lh /var/lib/docker
```

```bash
ls -lh /var/lib/docker/overlay2
```

Checking Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

If Docker data directory is migrated to `/data/docker`, the corresponding path may be:

```bash
/data/docker/overlay2
```

---

## Scenario 27: Why overlay2 Occupies Large Space

Common reasons:

```text
Too many mirrors.
Too many mirrors.
Packagings can write layers to a large number of files
Build too many caches
Old container uncleaned
Log file too big
volume The data is mistaken. overlay2 Problem
```

Troubleshooting:

```bash
docker system df
```

```bash
docker system df -v
```

```bash
du -sh /var/lib/docker/overlay2
```

Finding Large Directories:

```bash
du -h --max-depth=1 /var/lib/docker/overlay2 | sort -h
```

Finding Large Files:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

---

## Scenario 28: Do Not Manually Delete overlay2 Directories

High-risk operation:

```bash
rm -rf /var/lib/docker/overlay2/*
```

Risks:

```text
Docker Metadata damage
Mirror damage
The container cannot be activated.
volume Or a container layer anomaly.
Docker Service anomaly
```

Cleanup should prioritize using Docker commands:

```bash
docker system df
```

```bash
docker image prune
```

```bash
docker container prune
```

```bash
docker builder prune
```

```bash
docker system prune -a
```

Note:

```text
Impact ranges must be confirmed before production is cleared
```

---

## Section VII: Differences Between Image Layers, Container Layers, and Volumes

---

## Scenario 29: Relationship Between the Three

You can understand it as:

```text
Mirror Layer
→ Read-only
→ Multiple container sharing
→ From docker pull / docker build

Packaging Writing Layers
→ Writeable
→ Each container is independent
→ Deleting containers and disappearing.

volume
→ Independent data volume
→ Can be mounted by container
→ Default retention after container has been deleted
```

---

## Scenario 30: Where Should Data Be Placed

Suitable for image layer:

```text
Apply binary
Static File
Default Configuration
Run dependent
Basic tools
```

Suitable for container writable layer:

```text
Temporary documents
Short-life Cache
One-time Task Output
Temporary barriers files
```

Suitable for volume:

```text
Database data
Upload File
Business sustainability data
Log to keep
Intermediate Data Directory
```

---

## Scenario 31: Why Not Recommend Putting Database Data in Container Layer

If database data is written to the container layer:

```text
Packaging Delete
→ Data Delete
```

Moreover, writing intensively to the container writable layer is also unsuitable.

It is more recommended to use:

```bash
docker volume create mysql-data
```

```bash
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  mysql:8.0
```

---

## Section VIII: Relationship Between Container Processes and Host Processes

---

## Scenario 32: Container Processes Are Essentially Host Processes

Checking Container:

```bash
docker ps
```

Getting the PID of the container's main process on the host:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Checking Host Processes:

```bash
ps -p $(docker inspect -f '{{.State.Pid}}' ContainersID) -o pid,ppid,user,cmd
```

Understanding:

```text
The container is not a virtual machine.
The process in the container is the process on the host.
Just being... namespace / cgroup Separation and restrictions
```

---

## Scenario 33: Entering Container namespace for Troubleshooting /think

# Getting Container PID:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' ContainersID)
```

Viewing:

```bash
echo $PID
```

Using nsenter to enter network namespace:

```bash
nsenter -t $PID -n ip addr
```

Entering mount namespace:

```bash
nsenter -t $PID -m sh
```

Entering multiple namespaces:

```bash
nsenter -t $PID -n -m -p -u -i sh
```

Explanation:

```text
nsenter It's a bottom screener.
Prudence in production and use
```

If just for general troubleshooting, prefer using:

```bash
docker exec -it ContainersID /bin/sh
```

---

## IX. Summary of Differences Between Containers and Virtual Machines

---

## Scenario 34: Differences in Isolation Levels

Virtual Machine:

```text
Hardware virtualization
Every virtual machine has its own core.
Isolated.
Start slower
Resources cost more.
```

Container:

```text
Operating system level isolation
Shared host core
Segregation dependency namespace / cgroup / capability / seccomp Mechanisms
Start faster.
Resources cost less.
```

---

## Scenario 35: Differences in Security Boundaries

Container security boundaries are weaker than virtual machines.

Reason:

```text
Container shared host core
The kernel hole may affect container isolation.
Improper configuration of container privileges increases risk
privileged It will significantly reduce isolation.
Mount host sensitive directories are highly risky
```

High-risk example:

```bash
docker run -it --privileged -v /:/host alpine /bin/sh
```

Explanation:

```text
This type of operation is extremely risky.
The production environment is not to be used at random
```

---

## X. Common Underlying Issues and Troubleshooting

---

## Issue 1: Why does the container see PID 1?

Because of PID namespace isolation.

Inside the container:

```bash
ps aux
```

The main process might be:

```text
PID 1
```

On the host:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

You see the actual host PID.

---

## Issue 2: Why is 127.0.0.1 not the host inside the container?

Because of NET namespace isolation.

Inside the container:

```text
127.0.0.1
```

Represents the container's own loopback.

On the host:

```text
127.0.0.1
```

Represents the host's own loopback.

Therefore, when accessing host services from the container, use the host's reachable address, for example:

```text
docker0 Gateway Address
Host Intranet IP
host.docker.internal
```

Common Docker0 gateway on Linux:

```bash
ip addr show docker0
```

---

## Issue 3: Why do files disappear after deleting the container?

If the files are written to the container's writable layer:

```text
Packaging Delete
→ Packaging Writing Layer Deleting
→ File Missing
```

Solution:

```text
Enduring data use volume or bind mount
```

---

## Issue 4: Why can multiple containers share the same image?

Because the image layers are read-only and can be reused by multiple containers.

Each container only needs an additional writable layer.

Viewing the image:

```bash
docker images
```

Viewing the container:

```bash
docker ps -a
```

Viewing image history:

```bash
docker history Mirror Name
```

---

## Issue 5: Why modifying files in the container doesn't change the original image?

Because the container writes to its writable layer, not the image's read-only layer.

If you want to save the container's current state as a new image, you can use:

```bash
docker commit ContainersID myimage:v1
```

But in production, it's recommended to build using Dockerfile:

```bash
docker build -t myimage:v1 .
```

---

## Issue 6: Why can't the overlay2 directory be deleted randomly?

Because Docker's image layers, container layers, and metadata depend on these directories and Docker metadata relationships.

Direct deletion:

```bash
rm -rf /var/lib/docker/overlay2/*
```

May lead to:

```text
Mirror damage
Container damage
Docker Metadata inconsistent
Docker Service anomaly
```

Should use:

```bash
docker system df
```

```bash
docker image prune
```

```bash
docker container prune
```

```bash
docker builder prune
```

---

## Issue 7: Why do container resource limits not take effect or show anomalies?

Possible reasons:

```text
No resource limit set
cgroup driver Unusual
cgroup v1 / v2 Variance
Host kernel or Docker Version Limit
The container was configured abnormally when running
Kubernetes kubelet and runtime cgroup driver Inconsistencies
```

Viewing:

```bash
docker info | grep -i cgroup
```

```bash
mount | grep cgroup
```

```bash
stat -fc %T /sys/fs/cgroup
```

---

## XI. Common Command Summary

---

## Docker Information

View Docker information:

```bash
docker info
```

View storage driver:

```bash
docker info | grep "Storage Driver"
```

View Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

View cgroup information:

```bash
docker info | grep -i cgroup
```

---

## Namespace Related

View current shell namespace:

```bash
ls -l /proc/$$/ns
```

Get container host PID:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

View container process namespace:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' ContainersID)/ns
```

Use nsenter to view container network namespace:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' ContainersID)
```

```bash
nsenter -t $PID -n ip addr
```

---

## Cgroup Related

View cgroup version:

```bash
stat -fc %T /sys/fs/cgroup
```

View cgroup mounts:

```bash
mount | grep cgroup
```

View Docker cgroup driver:

```bash
docker info | grep -i "Cgroup Driver"
```

View container resource usage:

```bash
docker stats
```

Limit CPU:

```bash
docker run -d --cpus="1.5" nginx
```

Limit memory:

```bash
docker run -d -m 512m nginx
```

Limit process count:

```bash
docker run -d --pids-limit 100 nginx
```

Check for OOM:

```bash
docker inspect ContainersID | grep -i oom -A 5
```

---

## Overlay2 / Storage Related

View Docker usage:

```bash
docker system df
```

View detailed usage:

```bash
docker system df -v
```

View overlay2 directory:

```bash
ls -lh /var/lib/docker/overlay2
```

View overlay2 usage:

```bash
du -sh /var/lib/docker/overlay2
```

View large directories:

```bash
du -h --max-depth=1 /var/lib/docker/overlay2 | sort -h
```

Find large files:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

View image history:

```bash
docker history Mirror Name
```

---

## Volume Related

View volume:

```bash
docker volume ls
```

View volume details:

```bash
docker volume inspect volumeName
```

Create volume:

```bash
docker volume create mysql-data
```

Use volume:

```bash
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  mysql:8.0
```

---

## XII. One-Sentence Summary

Docker containers can be summarized as:

```text
namespace
→ Segregation View

cgroup
→ Resource constraints

overlay2
→ Manage mirror and container layers
```

Containers are not virtual machines:

```text
Container shared host core
Container process is essentially a host process
The container is through. namespace Looks like an independent environment.
The container is through. cgroup Resources restricted
The container is through. overlay2 Get a separate filesystem view
```

Namespace understanding:

```text
PID namespace
→ Isolation of process numbers in containers

NET namespace
→ Container network quarantine

MNT namespace
→ Packaging file system mounted view quarantine

UTS namespace
→ Hostname Separator

IPC namespace
→ Segregation of inter-process communication resources

USER namespace
→ User ID Segregation

CGROUP namespace
→ cgroup View Separator
```

Cgroup understanding:

```text
--cpus
→ Limits CPU

-m / --memory
→ Limit Memory

--pids-limit
→ Limit the number of processes

docker stats
→ View resource use
```

Overlay2 understanding:

```text
Mirror Layer
→ Read-only, reusable

Packaging Writing Layers
→ Each container is independent and disappears after it has been removed.

volume
→ Independent data volume, the container is deleted and retained by default

Copy while writing
→ Read the shared layer, copy it to the packaging writing layer
```

Production recommendations:

```text
Do not write persistent data on the packaging.
Database and intermediate data must be used volume Or external storage
Do not manually delete /var/lib/docker/overlay2
Check disk first. docker system df
Resource constraints depend on cgroupWatch it. cgroup driver
Packaging safety borders are weaker than virtual machines. Do not misuse them. privileged
Understood. namespace / cgroup / overlay2 After that,Docker The barrier will be clearer.
```