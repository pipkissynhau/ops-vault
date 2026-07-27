# 14-Docker Production Troubleshooting Case Studies

#Docker #Container Troubleshooting #Production Troubleshooting #containerd #Harbor #DockerCompose #Kubernetes #Logs #Network #Storage #Image Pulling #Operations and Maintenance

---

## Recommended Path

03-Container Technology/14-Docker Production Troubleshooting Case Studies.md

---

## I. Document Description

This document compiles common fault cases encountered in Docker production and near-production environments, focusing on:

- Containers exiting immediately after starting
- Containers restarting repeatedly
- Image pull failures
- Image build failures
- Inaccessibility after port mapping
- Services only bound to 127.0.0.1, making them inaccessible from outside
- Abnormal container DNS resolution
- Containers unable to access the external network
- Conflicts between Docker bridge IP ranges and the internal network
- Container logs filling up the disk
- `/var/lib/docker` taking up excessive space
- Volume mounting failures
- Risk of accidental data deletion with volumes
- Permission issues with docker cp commands
- Abnormal startup of Docker Compose services
- Issues where Compose depends_on does not match service availability
- HTTP/HTTPS errors with Harbor
- Successful docker login but failed K8s image pull
- Unauthorized authentication failures
- Rejected image pushes
- Misreading containerd namespace, resulting in "image not found" errors
- Abnormal container resource usage
- Container OOM/memory不足 issues
- Privileged/capability permission problems
- Troubleshooting unhealthy healthchecks

The goal is:

- To quickly identify the issue based on the symptoms
- To select the correct commands for troubleshooting
- To distinguish between Docker, containerd, and Kubernetes
- To avoid accidental data deletion and incorrect repairs
- To establish a closed-loop for production troubleshooting

---

## II. General Docker Troubleshooting Approach

When troubleshooting Docker issues, do not immediately restart services.

It is recommended to first confirm the following in order:

```text
Whether the container exists
→ Whether the container is running
→ Whether there are any error messages in the container logs
→ Whether the container startup parameters are correct
→ Whether ports have been mapped
→ Whether the host machine is listening on those ports
→ Whether the network is functioning properly
→ Whether DNS is working correctly
→ Whether volumes have been mounted successfully
→ Whether the image is correct
→ Whether there is a shortage of resources
→ Whether there are any permission restrictions
```

Common basic commands:

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker logs containerID
```

```bash
docker logs --tail 100 -f containerID
```

```bash
docker inspect containerID
```

```bash
docker stats
```

```bash
docker system df
```

```bash
docker info
```

---

## III. Case 1: Containers Exiting Immediately After Starting

---

## Phenomenon

When executing:

```bash
docker run imageName
```

The container exits shortly after starting.

Checking the containers:

```bash
docker ps
```

No running containers are displayed.

Checking all containers:

```bash
docker ps -a
```

The container status is shown as:

```text
Exited
```

---

## Common Causes

```text
The main process exits immediately after completion
Incorrect CMD/ENTRYPOINT settings
Application startup failure
Configuration file errors
Dependent services are unavailable
Insufficient permissions
File not found
Port conflicts
```

---

## Troubleshooting Steps

Check all containers:

```bash
docker ps -a
```

View the logs:

```bash
docker logs containerID
```

Review recent logs:

```bash
docker logs --tail 100 containerID
```

Inspect the container details:

```bash
docker inspect containerID
```

Pay special attention to:

```text
State.ExitCode
State.Error
Config Cmd
Config.Entrypoint
Mounts
Env
```

---

## Typical Example

If the Dockerfile contains:

```dockerfile
CMD ["echo", "hello"]
```

The container will execute `echo hello` once and then exit, as the main process terminates afterward.

This is normal behavior.

For long-running services, a foreground main process is necessary.

Nginx recommends:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

---

## Handling Approach

```text
Check the logs
→ Check the ExitCode
→ Verify the startup command
→ Review the configuration file
→ Check dependent services
→ Ensure the main process is running in the foreground
```

---

## IV. Case 2: Containers Restarting Repeatedly

---

## Phenomenon

When checking containers:

```bash
docker ps -a
```

The status shows:

```text
Restarting
```

Or the container keepsFROM golang:1.23 AS build

RUN go build -o app main.go


FROM alpine:3.20

COPY --from=builder /src/app /app/app
```## Case 13: Excessive Usage of /var/lib/docker

---

## Symptoms

Check the disk usage:

```bash
df -h
```

Check the Docker directory usage:

```bash
du -sh /var/lib/docker
```

The usage is very high.

---

## Common Causes

```text
Too many images
Too many stopped containers
Too many volumes
Too many build caches
Large container logs
Old image versions not cleaned up
```

---

## Troubleshooting Steps

Check Docker usage:

```bash
docker system df
```

View detailed usage:

```bash
docker system df -v
```

List images:

```bash
docker images
```

List all containers:

```bash
docker ps -a
```

List volumes:

```bash
docker volume ls
```

Check the Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

---

## Cleanup Commands

Clean up unused images:

```bash
docker image prune
```

Clean up stopped containers:

```bash
docker container prune
```

Clean up unused volumes:

```bash
docker volume prune
```

Clean up unused resources:

```bash
docker system prune -a
```

---

## Notes

Be cautious when executing these commands in a production environment:

```bash
docker volume prune
```

Reason:

```text
Volume contents may include database data, uploaded files, or persistent business data.
```

---

## Case 14: Volume Mounting Issues

---

## Symptoms

The container starts successfully, but application data is not saved as expected, or host files are not visible inside the container.

---

## Common Causes

```text
The host path does not exist.
The host directory permissions are incorrect.
The volume name is misspelled.
The path inside the container is incorrect.
Bind mount overwrites original files in the image.
The container running user lacks the necessary permissions.
SELinux/AppArmor restrictions.
```

---

## Troubleshooting Steps

Check container details:

```bash
docker inspect containerID
```

View mounting information:

```bash
docker inspect containerID | grep Mounts -A 30
```

List volumes:

```bash
docker volume ls
```

View volume details:

```bash
docker volume inspect volumeName
```

If it's a bind mount, check the host directory:

```bash
ls -ld /data/app
```

Check permissions:

```bash
ls -lah /data/app
```

---

## Resolution Steps

```text
Confirm the mount source path.
→ Confirm the target path inside the container.
→ Check permissions.
→ Verify the container running user.
→ Ensure original files in the image are not overwritten.
```

---

## Case 15: Risk of Accidental Volume Data Deletion

---

## High-Risk Commands

```bash
docker volume prune
```

```bash
docker compose down -v
```

---

## Risks

These commands may delete data volumes that are not currently in use by any containers.

If the volume contains:

```text
MySQL data
PostgreSQL data
Redis persistence files
Application uploaded files
Business configuration files
```

Deletion could result in data loss.

---

## Troubleshooting and Confirmation

List volumes:

```bash
docker volume ls
```

View volume details:

```bash
docker volume inspect volumeName
```

Check container mounts:

```bash
docker inspect containerID | grep Mounts -A 30
```

---

## Production Recommendations

```text
Always confirm the value of data before cleaning up volumes.
Make sure database volumes have backups.
Never execute `docker compose down -v` without understanding the potential impact.
```

---

## Case 16: Errors with docker cp Due to Incorrect Paths or Permissions

---

## Symptoms

Attempting to copy files using:

```bash
docker cp containerID:/path /hostPath
```

Results in failure.

---

## Common Causes

```text
Incorrect container ID or name.
The path inside the container does not exist.
The target path on the host does not exist.
Insufficient permissions.
The path is not specified as an absolute path.
The container has already been deleted.
```

---

## Troubleshooting Steps

Check containers:

```bash
docker ps -a
```

Confirm the path inside the container:

```bash
docker exec -it containerID /bin/sh
```

```bash
ls -lah /etc/nginx
```

Try copying files:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

Check the host directory:

```bash
ls -ld /root/
```

After copying, check permissions:

```bash
ls -lh /root/nginx.conf
```

Adjust permissions if necessary:

```bash
chown root:root /root/nginx.conf
```

```bash
chmod 644 /root/nginx.conf
```

---

## Notes

Files can also be copied from a stopped container as## Phenomenon

When using `docker pull` or `docker push`, an error occurs:

```text
http: server gave HTTP response to HTTPS client
```

---

## Cause

```text
By default, the client accesses repositories via HTTPS.
However, Harbor actually uses HTTP.
```

---

## Docker Solutions

Edit:

```bash
vi /etc/docker/daemon.json
```

Configure:

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

Restart:

```bash
systemctl daemon-reload
```

```bash
systemctl restart docker
```

Verify:

```bash
docker info
```

---

## Containerd Solutions

Create a directory:

```bash
mkdir -p /etc/containerd/certs.d/10.0.0.10:8090/
```

Edit:

```bash
vi /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Content:

```toml
server = "http://10.0.0.10:8090"

[host."http://10.0.0.10:8090"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

Check `config_path`:

```bash
grep -n "config_path" /etc/containerd/config.toml
```

It should contain:

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

Restart containerd:

```bash
systemctl restart containerd
```

---

## Case 21: Successful Docker Login, but Failed to Pull Images from K8s

---

## Phenomenon

On the node, executing `docker login 10.0.0.10:8090` is successful.

However, trying to pull images for Kubernetes Pods fails.

---

## Core Reason

```text
When a K8s node uses containerd,
Pod image pulls are handled by containerd,
not by the Docker login state.
```

Therefore:

```text
Successful Docker login does not guarantee that a K8s Pod can pull private images.
```

---

## Troubleshooting Steps

Check Pod events:

```bash
kubectl describe pod PodName -n Namespace
```

View containerd configuration:

```bash
cat /etc/containerd/config.toml
```

Check Harbor `hosts.toml`:

```bash
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Test with `crictl`:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Test with `ctr`:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

Check Secrets:

```bash
kubectl get secret -n Namespace
```

View Pod YAML:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

---

## Correct Approach

Create an `imagePullSecret`:

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=10.0.0.10:8090 \
  --docker-username=admin \
  --docker-password='Harbor12345' \
  --docker-email(admin@example.com \
  -n default
```

Reference it in the Pod YAML:

```yaml
imagePullSecrets:
  - name: harbor-secret
```

Note:

```text
The Secret must be in the same namespace as the Pod.
```

---

## Case 22: Unauthorized: Authentication Required

---

## Phenomenon

When trying to pull or push images, the following error occurs:

```text
unauthorized: authentication required
```

---

## Common Reasons

```text
Not logged in to Harbor.
Incorrect account credentials.
Lack of project permissions.
The Harbor project is private.
Wrong image path.
K8s does not have `imagePullSecrets` configured.
Incorrect Secret namespace.
```

---

## Docker Troubleshooting

Log in:

```bash
docker login 10.0.0.10:8090
```

Pull images:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

Check authentication file:

```bash
cat ~/.docker/config.json
```

Note:

```text
~/.docker/config.json may contain sensitive information; do not disclose it casually.
```

---

## Kubernetes Troubleshooting

Check Secrets:

```bash
kubectl```bash
docker inspect containerID
```

Focus on:

```text
State.OOMKilled
State.ExitCode
HostConfig.Memory
```

You can use:

```bash
docker inspect containerID | grep -i oom -A 5
```

To view the host kernel logs:

```bash
dmesg | grep -i kill
```

Or:

```bash
journalctl -k | grep -i oom
```

---

## Common Causes

```text
The container's memory limit is too low.
Application has a memory leak.
Sudden high traffic volume.
Unreasonable memory parameters for runtime environments like JVM/Node.js.
Insufficient overall host memory.
```

---

## Troubleshooting Steps

```text
Confirm if it was OOMKilled.
→ Check application logs.
→ Verify the container's memory limit.
→ Adjust application memory settings.
→ Modify the container's memory limit.
→ Conduct a capacity assessment.
```

---

## Case 25: Insufficient Permissions Inside the Container

---

## Symptoms

When executing commands inside the container, errors occur such as:

```text
Operation not permitted
```

Or:

```text
Permission denied
```

---

## Common Reasons

```text
The container is running under a non-root user.
Insufficient file permissions.
Lack of certain Linux capabilities.
Devices are not mapped into the container.
SECCOMP/AppArmor/SELinux restrictions.
No privileged permissions available.
```

---

## Troubleshooting Steps

Check the container user:

```bash
docker exec -it containerID id
```

Verify file permissions:

```bash
docker exec -it containerID ls -lah /path
```

Inspect the container configuration:

```bash
docker inspect containerID
```

---

## Common Capability Scenarios

To manage network resources:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

To access raw network packets:

```bash
docker run -d --cap-add NET_RAW nginx
```

For process debugging:

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

Device mapping:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

To use privileged mode:

```bash
docker run -d --privileged nginx
```

---

## Production Recommendations

```text
Don't immediately switch to privileged mode whenever you encounter permission issues.
First, determine which specific permissions are missing.
Try using `--cap-add` first.
Use `--device` when devices are needed.
Only consider `--privileged` as a last resort.
```

---

## Case 26: Inability to Ping Within the Container or Permission Issues

---

## Symptoms

When attempting to execute:

```bash
ping 8.8.8.8
```

within the container, an error is displayed:

```text
Operation not permitted
```

---

## Possible Reasons

```text
Lack of `NET_RAW` capability.
The image does not contain a `ping` command.
The network is unavailable.
Security policies are restricting access.
```

---

## Troubleshooting Steps

Enter the container:

```bash
docker exec -it containerID /bin/sh
```

Check if the `ping` command is available:

```bash
which ping
```

Verify the routing configuration:

```bash
ip route
```

Perform a temporary test:

```bash
docker run --rm -it --cap-add NET_RAW busybox /bin/sh
```

Then attempt to ping again:

```bash
ping 8.8.8.8
```

---

## Note

```text
If `ping` fails, it doesn't necessarily mean that TCP communication is also blocked.
In some environments, ICMP is disabled, but TCP access is still possible.
```

You can also use:

```bash
curl -I http://example.com
```

Or:

```bash
nc -vz targetIP port
```

---

## Case 27: Inability to Access Host Services from Within the Container

---

## Symptoms

A service on the host is listening on `127.0.0.1:3306`, but attempts to access it from within the container fail.

---

## Reason

Within the container, `127.0.0.1` refers to the container itself, not the host.

---

## Troubleshooting Steps

On the host, check the listening port:

```bash
ss -tunlp
```

Verify the Docker bridge gateway address:

```bash
ip addr show docker0
```

Inside the container, check the routing configuration:

```bash
ip route
```

---

## Common Ways to Access Services

Use the host's `docker0` address:

```text
172.17.0.1
```

Access from within the container:

```bash
curl http://172.17.0.## Storage and Log Troubleshooting

```bash
df -h
```

```bash
du -sh /var/lib/docker
```

```bash
docker system df -v
```

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

---

## Compose Troubleshooting

```bash
docker compose config
```

```bash
docker compose ps
```

```bash
docker compose logs -f
```

```bash
docker compose logs -f service_name
```

```bash
docker compose exec service_name /bin/sh
```

```bash
docker compose down
```

---

## Harbor/Image Troubleshooting

```bash
docker login 10.0.0.10:8090
```

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

```bash
docker info
```

```bash
cat ~/.docker/config.json
```

---

## containerd/K8s Node Troubleshooting

```bash
systemctl status containerd
```

```bash
journalctl -u containerd -f
```

```bash
crictl ps -a
```

```bash
crictl images
```

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

```bash
ctr -n k8s.io images ls
```

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

```bash
nerdctl --namespace k8s.io images ls
```

```bash
kubectl describe pod Pod_name -n namespace
```

```bash
journalctl -u kubelet -f
```

---

## 34. Troubleshooting Priority Recommendations

Recommended priorities for production troubleshooting:

```text
1. First, determine the scope of impact.
2. Preserve all relevant on-site information.
3. Then check logs and status.
4. Avoid deleting containers, volumes, or images prematurely.
5. Do not restart Docker blindly.
6. Do not directly execute `docker system prune -a`.
7. Do not directly perform `docker compose down -v`.
8. Do not assume privileged access from the beginning.
9. Identify the root cause before taking any corrective actions.
10. Verify and document the results after repairs.
```

---

## 35. One-Sentence Summary

The core of Docker production troubleshooting is:

```text
Check status → Analyze logs → Review configurations → Examine networks → Verify storage → Check permissions → Assess resources → Verify image repositories → Evaluate runtime boundaries.
```

Basic troubleshooting steps include:

```text
docker ps -a → docker logs → docker inspect → docker stats → docker system df
```

For network issues:

```text
docker port → ss -tunlp → ip route → iptables → docker network inspect → DNS/routing tests within containers
```

For storage issues:

```text
df -h → docker system df → du -sh /var/lib/docker → Check container log files → Verify volumes and mounts
```

For image pull issues:

```text
docker pull → docker login → Harbor project and permissions → HTTP/HTTPS configuration → containerd hosts.toml → crictl/ctr tests → K8s imagePullSecrets
```

For Compose issues:

```text
docker compose config → docker compose ps → docker compose logs -f → Health checks → Dependencies → Environment variables/volumes/network configurations
```

Production principles include:

```text
Observe before acting. Back up before cleaning up. Identify the issue before restarting. Use minimal privileges whenever possible. Confirm data value before deleting volumes. Verify runtime conditions before assessing Docker/containerd/K8s boundaries.
```