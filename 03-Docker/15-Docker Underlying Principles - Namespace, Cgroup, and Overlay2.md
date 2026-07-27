# 15-Docker Underlying Principles: Namespace, Cgroup, and Overlay2

#Docker #Container Principles #namespace #cgroup #overlay2 #UnionFS #Copy-on-Write #Image Layering #Operation and Maintenance #Underlying Principles

---

## Recommended Path

03-Container Technology/15-Docker Underlying Principles: Namespace, Cgroup, and overlay2.md

---

## I. Document Description

This article outlines three core concepts in Docker's underlying principles:

- namespace
- cgroup
- overlay2

Key points include:

- Why containers are not virtual machines
- How containers achieve isolation
- What namespace does
- What cgroup does
- What overlay2 does
- The relationship between container processes and host processes
- The relationship between container file systems and image layers
- Why images are layered
- Why containers have a writable layer
- What copy-on-write is
- Why volume data remains after the container is deleted
- Why overlay2 consumes disk space
- Common underlying troubleshooting commands

The goal is:

To understand the underlying components of Docker containers

→ To explain the differences between containers and virtual machines

→ To comprehend the isolation mechanism of namespace

→ To understand the resource limitation mechanism of cgroup

→ To understand overlay2's image layering and copy-on-write technology

→ To correlate Docker command behaviors with Linux underlying mechanisms

---

## II. Containers Are Not Virtual Machines

Containers are not virtual machines.

Virtual machines typically consist of:

```text
Host hardware
→ Host operating system
→ Hypervisor
→ Virtual machine operating system
→ Application processes
```

Containers, on the other hand, consist of:

```text
Host hardware
→ Host Linux kernel
→ Docker / containerd
→ Container processes
```

The key difference is:

```text
Virtual machines have their own complete operating system kernel

Containers share the host kernel
```

This makes containers more lightweight:

```text
Fast to start up
Low resource consumption
Easy image distribution
More suitable for microservices and cloud-native deployment
```

However, it's also important to note that:

```text
Container isolation is weaker than that of virtual machines
Container security depends on the host kernel's capabilities
Host kernel vulnerabilities can affect container security
```

---

## III. Docker Containers' Three Underlying Components

Docker containers can be understood as a combination of three Linux features:

```text
namespace
→ Achieves isolation

cgroup
→ Limits resources and provides statistics

overlay2
→ Handles image layering and the container's writable layer
```

In simple terms:

```text
Namespace makes the container "act like an independent machine"

Cgroups control how much resource a container can use

Overlay2 creates an independent file system view for the container
```

---

## IV. Namespace: The Foundation of Container Isolation

---

## Scenario 1: What is a Namespace?

A namespace is an isolation mechanism provided by the Linux kernel.

It can be understood as:

```text
Namespace = Creating an isolated system view for processes
```

For example, within a container:

```text
The container sees its own process list
The container sees its own hostname
The container sees its own network devices
The container sees its own mount points
The container sees its own IPC resources
```

But essentially:

```text
Processes inside the container are still host processes
They are just placed in different namespaces
```

---

## Scenario 2: Common Types of Namespaces

Common namespaces include:

```text
PID namespace
→ Isolates process IDs

NET namespace
→ Isolates the network stack

MNT namespace
→ Isolates mount points

UTS namespace
→ Isolates hostname and domain names

IPC namespace
→ Isolates inter-process communication resources

USER namespace
→ Isolates user and user group IDs

CGROUP namespace
→ Isolates cgroup views
```

---

## Scenario 3: PID Namespace

The PID namespace is used to isolate process IDs.

Viewing processes inside a container:

```bash
ps aux
```

You might see that the main container process has a PID of:

```text
1
```

But on the host, it is just an ordinary host process.

Viewing processes inside the container:

```bash
docker exec -it containerID ps aux
```

Viewing Docker-related processes on the host:

```bash
ps aux | grep container_process_keyword
```

Understanding:

```text
PID 1 inside the container
→ Represents process 1 from the container's perspective

Host PID
→ Represents the actual process ID on the host
```

---

## Scenario 4: Why Does a Container Shut Down When Its Main Process Ends?

Containers usually rely on a foreground main process.

For example:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

If the main process exits, the container will also shut down.

Incorrect example:

```bash
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' containerID)/ns
``````bash
docker inspect -f '{{.State.Pid}}' 容器ID
``````bash
docker inspect -f "{{.State.Pid}}' container ID
```

To view the container process namespace:

```bash
ls -l /proc/$(docker inspect -f "{{.State.Pid}}' container ID)/ns
```

Use nsenter to view the container network namespace:

```bash
PID=$(docker inspect -f "{{.State.Pid}}' container ID)
nsenter -t $PID -n ip addr
```

---

## CGroup Related Commands

To check the CGroup version:

```bash
stat -fc %T /sys/fs/cgroup
```

To view CGroup mounts:

```bash
mount | grep cgroup
```

To check the Docker CGroup driver:

```bash
docker info | grep -i "Cgroup Driver"
```

To view container resource usage:

```bash
docker stats
```

To limit CPU usage:

```bash
docker run -d --cpus="1.5" nginx
```

To limit memory usage:

```bash
docker run -d -m 512m nginx
```

To limit the number of processes:

```bash
docker run -d --pids-limit 100 nginx
```

To check for OOM (Out of Memory):

```bash
docker inspect container ID | grep -i oom -A 5
```

---

## Overlay2 / Storage Related Commands

To view Docker disk usage:

```bash
docker system df
```

For detailed usage information:

```bash
docker system df -v
```

To list the overlay2 directories:

```bash
ls -lh /var/lib/docker/overlay2
```

To check the overlay2 storage usage:

```bash
du -sh /var/lib/docker/overlay2
```

To view large directories in overlay2:

```bash
du -h --max-depth=1 /var/lib/docker/overlay2 | sort -h
```

To find large files in overlay2:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

To view image history:

```bash
docker history image_name
```

---

## Volume Related Commands

To list volumes:

```bash
docker volume ls
```

To check volume details:

```bash
docker volume inspect volume_name
```

To create a volume:

```bash
docker volume create mysql-data
```

To use a volume in a container:

```bash
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  mysql:8.0
```

---

## Summary

The underlying principles of Docker containers can be summarized as follows:

- **Namespace**: Provides isolation between different components within a container.
- **CGroup**: Used to limit resources such as CPU and memory usage.
- **Overlay2**: Manages the image layer and the container’s writable data layer, ensuring data persistence.

Docker containers do not act like traditional virtual machines because they:

- Share the host kernel, meaning container processes are essentially host processes.
- Use namespaces to present a separate environment for each container.
- Rely on CGroups to control resource allocation.
- Utilize overlay2 to provide an independent file system view for each container.

Understanding namespace concepts is crucial, as they enable:

- Isolation of process IDs (`PID namespace`).
- Separation of container networks (`NET namespace`).
- isolation of file system mounts (`MNT namespace`).
- Differentiation between host and container names (`UTS namespace`).
- Protection of inter-process communication resources (`IPC namespace`).
- Maintenance of unique user IDs (`USER namespace`).
- Customization of CGroup settings (`CGROUP namespace`).

CGroup settings include options like `--cpus` to limit CPU usage, `-m /` or `--memory` to restrict memory allocation, and `--pids-limit` to control the number of processes. The `docker stats` command provides insights into container resource consumption.

As for overlay2, it consists of an immutable image layer and a writable data layer for each container. This design ensures that data is persistent across container restarts. Volumes provide additional flexibility by allowing independent data storage that is retained even after the container is deleted. The write-through mechanism in overlay2 means that reads are shared from the image layer, while writes are copied to the container’s writable data layer.

In production environments, it is important to:

- Avoid storing persistent data in the container’s writable layer.
- Use volumes or external storage for critical data like databases and middleware.
- Not manually delete the `/var/lib/docker/overlay2` directory.
- Always use `docker system df` to diagnose disk issues.
- Pay close attention to the CGroup driver since it affects resource management.
- Be cautious with using privileged permissions in containers, as their security boundaries are weaker than those of virtual machines.
-