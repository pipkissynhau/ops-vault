# 14-Docker Production Troubleshooting Case Studies

#Docker #PackagingBarriers #ProductionBarriers #containerd #Harbor #DockerCompose #Kubernetes #Log #Network #Storage #MirrorPull #Transport

---

## Recommended Path

03-Container Technology/14-Docker Production Troubleshooting Case Studies.md

---

## One: Document Description

This document compiles common fault cases in Docker production and pre-production environments, focusing on:

- Container exits immediately after startup
- Container repeated restarts
- Image pull failure
- Image build failure
- Port mapping access failure
- Service bound only to 127.0.0.1 causing external unavailability
- Container DNS resolution anomalies
- Container inability to access external network
- Docker bridge network segment conflict with internal network
- Container log filling disk
- `/var/lib/docker` occupying excessive space
- Volume mount anomalies
- Risk of volume data deletion
- docker cp path and permission issues
- Docker Compose service startup anomalies
- Compose depends_on not equal to service availability
- Harbor HTTP/HTTPS error
- docker login success but K8s image pull failure
- unauthorized authentication failure
- Image push rejection
- containerd namespace misreading causing "image not found"
- Container resource occupation anomalies
- Container OOM/memory insufficiency
- privileged/capability permission issues
- healthcheck unhealthy investigation

The goal is:

- Quickly locate the direction by phenomenon
- Choose the correct command for investigation
- Distinguish the boundary between Docker, containerd, and Kubernetes
- Avoid data deletion and incorrect fixes
- Form a production troubleshooting closed loop

---

## Two: Docker Troubleshooting General Approach

Do not restart services immediately when encountering Docker issues.

Suggested steps in order:

```text
Existence of container
→ whether the container is running
→ Wrong container log
→ Is the container starter parameter correct?
→ Port Map
→ Is the host listening?
→ Is the network normal?
→ DNS Is it normal?
→ volume Whether to mount
→ Is the mirror correct?
→ Inadequate resources
→ Permission restricted
```

Common basic commands:

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker logs ContainersID
```

```bash
docker logs --tail 100 -f ContainersID
```

```bash
docker inspect ContainersID
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

## Three: Case 1: Container Exits Immediately After Startup

---

## Phenomenon

Execute:

```bash
docker run Mirror Name
```

The container starts and exits quickly.

Check container:

```bash
docker ps
```

No running containers are visible.

Check all containers:

```bash
docker ps -a
```

You can see container status similar to:

```text
Exited
```

---

## Common Causes

```text
When the main process is completed, exit.
CMD / ENTRYPOINT Wrong
Application startup failed
Profile Error
Reliance on services not available
Insufficient Permissions
File does not exist
Port Conflict
```

---

## Troubleshooting Steps

Check all containers:

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

Check container details:

```bash
docker inspect ContainersID
```

Focus on:

```text
State.ExitCode
State.Error
Config.Cmd
Config.Entrypoint
Mounts
Env
```

---

## Typical Example

If Dockerfile contains:

```dockerfile
CMD ["echo", "hello"]
```

The container will execute `echo hello` once, then the main process ends, and the container naturally exits.

This is normal behavior.

Long-running services must have a foreground main process.

Nginx recommendation:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

---

## Handling Approach

```text
Read the log.
→ Look. ExitCode
→ Look at the start-up order.
→ Look at the profile.
→ Looking at services.
→ Confirm whether the main process is running at the front desk
```

---

## Four: Case 2: Container Repeated Restart

---

## Phenomenon

Check container:

```bash
docker ps -a
```

Status similar to:

```text
Restarting
```

Or the container keeps exiting and restarting.

---

## Common Causes

```text
Application startup failed
Configure Error
Starting command error
Reliance on services not available
Port Conflict
Insufficient memory due OOM
Health check failure
restart The strategy has led to a reboot.
```

---

## Troubleshooting Steps

Check status:

```bash
docker ps -a
```

Check logs:

```bash
docker logs ContainersID
```

Real-time log viewing:

```bash
docker logs -f ContainersID
```

Check last 100 lines:

```bash
docker logs --tail 100 ContainersID
```

Check container details:

```bash
docker inspect ContainersID
```

Check resources:

```bash
docker stats
```

---

## Key Checks

Check if restart policy is set:

```bash
docker inspect ContainersID | grep -A 10 RestartPolicy
```

Common policies:

```text
no
on-failure
always
unless-stopped
```

---

## Handling Approach

```text
Don't be blind. docker restart
→ Look first. logs
→ Look again. inspect
→ Let's judge whether it's application, allocation, dependence or resources.
```

If the repeated restart is caused by the restart policy, first resolve the actual application error.

---

## Five: Case 3: Image Pull Failure

---

## Phenomenon

Execute:

```bash
docker pull nginx:latest
```

Or:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

Failure occurs.

---

## Common Causes

```text
The mirror name is wrong.
tag does not exist
Network's impossible.
DNS Parsing failed
Private warehouse not login
Harbor Project does not exist
Harbor Insufficient Permissions
HTTP Harbor Not matched insecure registry
```

---

## Troubleshooting Steps

Confirm image name:

```bash
docker pull nginx:latest
```

Confirm network:

```bash
ping 10.0.0.10
```

Check DNS:

```bash
nslookup registry-1.docker.io
```

Or:

```bash
dig registry-1.docker.io
```

Login to private registry:

```bash
docker login 10.0.0.10:8090
```

Pull Harbor image:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

---

## HTTP Harbor Special Troubleshooting

If error:

```text
http: server gave HTTP response to HTTPS client
```

Indicates client is accessing via HTTPS, but Harbor is actually HTTP.

Docker configuration:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "insecure-registries": ["10.0.0.10:8090"]
}
```

Restart Docker:

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

Check if contains:

```text
Insecure Registries:
  10.0.0.10:8090
```

---

## Six: Case 4: Dockerfile Build Failure

---

## Phenomenon

Execute:

```bash
docker build -t myapp:v1 .
```

Build failure.

Common error:

```text
COPY failed
file not found
apt-get update failed
permission denied
failed to solve
```

---

## Common Causes

```text
Dockerfile Path error
Build Context Error
COPY File does not exist
.dockerignore File excluded
Basic mirror pull failed
Software source not available
Network or DNS It's not working.
Inadequate document permissions
Multistage COPY --from Wrong
```

---

## Troubleshooting Steps

Confirm current directory:

```bash
pwd
```

Check files:

```bash
ls -lah
```

Check Dockerfile:

```bash
cat Dockerfile
```

Check `.dockerignore`:

```bash
cat .dockerignore
```

Execute build:

```bash
docker build -t myapp:v1 .
```

Build without cache:

```bash
docker build --no-cache -t myapp:v1 .
```

---

## COPY Failed Troubleshooting

Error example:

```dockerfile
COPY app.jar /app/app.jar
```

But current directory lacks `app.jar`.

Check:

```bash
ls -lh app.jar
```

If file is excluded by `.dockerignore`, it will also cause copy failure.

---

## Multi-stage COPY --from Troubleshooting

Error example:

```dockerfile
FROM golang:1.23 AS build

RUN go build -o app main.go


FROM alpine:3.20

COPY --from=builder /src/app /app/app
```

Issue:

```text
Define Phase Name build
The reference stage is: builder
```

Correct syntax:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /src

COPY . .

RUN go build -o app main.go


FROM alpine:3.20

WORKDIR /app

COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

---

## Seven: Case 5: Port Mapping Access Failure

---

## Phenomenon

Start container:

```bash
docker run -d -p 8080:80 nginx
```

Access:

```text
http://HostIP:8080
```

Access failure.

---

## Common Causes

```text
The container is not running
PortMap Error
The container service is not listening.
The host port is not listening.
Firewall blocked.
Security team not released.
Services only listen. 127.0.0.1
Visits IP Wrong.
```

---

## Troubleshooting Steps

Check container:

```bash
docker ps
```

Check port mapping:

```bash
docker port ContainersID
```

Check host listening:

```bash
ss -tunlp
```

Check logs:

```bash
docker logs ContainersID
```

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Check listening inside container:

```bash
ss -tunlp
```

---

## Access Testing

Host machine local testing:

```bash
curl -I http://127.0.0.1:8080
```

Other machine testing:

```bash
curl -I http://HostIP:8080
```

---

## Handling Approach

```text
docker ps See if the container is running.
→ docker port See if the map is right.
→ ss -tunlp See if the host is listening.
→ docker logs See if application is wrong
→ Firewall / Security team confirm release.
```

---

## VIII. Case 6: Port only binds to 127.0.0.1, external access fails

---

## Phenomenon

Startup command:

```bash
docker run -d -p 127.0.0.1:8080:80 nginx
```

Host machine can access:

```bash
curl http://127.0.0.1:8080
```

External machine access fails.

---

## Cause

```text
127.0.0.1 Just listen to the host's address.
External machines cannot access
```

This is expected behavior, not a fault.

---

## Check Listening

```bash
ss -tunlp | grep 8080
```

May see:

```text
127.0.0.1:8080
```

---

## Resolution

If you want external access, use:

```bash
docker run -d -p 8080:80 nginx
```

Equivalent to:

```text
0.0.0.0:8080 -> Containers:80
```

If you only bind to a specific internal IP:

```bash
docker run -d -p 10.0.0.10:8080:80 nginx
```

---

## IX. Case 7: Container DNS Resolution Abnormality

---

## Phenomenon

Container can access IP, but cannot resolve domain names.

For example:

```bash
ping 8.8.8.8
```

Works.

But:

```bash
ping www.baidu.com
```

Fails.

---

## Common Causes

```text
Containers DNS Configure Abnormal
Host DNS Unusual
Docker daemon DNS Configure Error
Intranet DNS Unattainable.
Firewall restrictions DNS
/etc/resolv.conf Incorrect
```

---

## Troubleshooting Steps

Check container DNS:

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Test resolution:

```bash
nslookup www.baidu.com
```

Or:

```bash
getent hosts www.baidu.com
```

Check host machine DNS:

```bash
cat /etc/resolv.conf
```

---

## Configure Docker DNS

Edit:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Note:

```text
Modify Docker daemon DNS After that, re-created packagings are usually required to be fully effective.
```

---

## X. Case 8: Container Cannot Access Internet

---

## Phenomenon

Container fails to access internet:

```bash
curl http://example.com
```

Fails.

---

## Common Causes

```text
The host itself cannot access the Internet.
Containers DNS Unusual
iptables The transmission rule is abnormal.
Docker bridge Network anomaly
Default route anomaly
Firewall restrictions
Cloud Safety Unit Restrictions
Cannot configure proxy environment
```

---

## Troubleshooting Steps

Host machine testing:

```bash
curl -I http://example.com
```

Check host machine routing:

```bash
ip route
```

Check iptables:

```bash
iptables -L -n
```

Check NAT table:

§

Check Docker network:

```bash
docker network ls
```

```bash
docker network inspect bridge
```

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Container routing check:

```bash
ip route
```

Container testing:

```bash
ping 8.8.8.8
```

```bash
nslookup www.baidu.com
```

---

## Handling Approach

```text
Make sure the host gets access. Network
→ Let's make sure the container is ready. ping IP
→ Reconfirm. DNS
→ Look again. Docker bridge
→ Look again. iptables / NAT / Firewall
```

---

## XI. Case 9: Docker bridge Network Segment Conflicts with Internal Network

---

## Phenomenon

Container fails to access certain internal network addresses, but other addresses work normally.

For example:

```text
Container access 172.17.x.x Innernet failed
Host access normal
Other extranet access is normal
```

---

## Common Causes

Docker's default bridge network segment may conflict with company internal network, cloud VPC, or VPN network segments.

Common Docker default network segments:

```text
172.17.0.0/16
```

---

## Troubleshooting Steps

Check Docker bridge:

```bash
docker network inspect bridge
```

Check host machine routing:

```bash
ip route
```

Enter container to check routing:

```bash
docker exec -it ContainersID ip route
```

Confirm company internal network/VPC/VPN network segments:

```text
IDC Intranet
Clouds VPC Network
VPN Network
Kubernetes Pod Network
Kubernetes Service Network
```

---

## Modify Docker Default Network Segment

Edit:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "bip": "172.31.0.1/24",
  "default-address-pools": [
    {
      "base": "172.32.0.0/16",
      "size": 24
    }
  ]
}
```

Restart Docker:

```bash
systemctl restart docker
```

Note:

```text
Modify Docker The network may affect existing containers.
The production environment requires maintenance windows
```

---

## XII. Case 10: Container Log Fills Disk

---

## Phenomenon

Host machine disk alert.

Check:

```bash
df -h
```

Find system disk full.

Docker directory occupies a lot:

```bash
du -sh /var/lib/docker
```

Container logs are large:

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

---

## Common Causes

When Docker uses the default `json-file` log driver, container standard output logs are written to the host file. Without log size limits, logs may grow infinitely.

---

## Troubleshooting Steps

Check Docker disk usage:

```bash
docker system df
```

Check container log files:

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

Find large files:

```bash
find /var/lib/docker -type f -size +500M -exec ls -lh {} \;
```

---

## Configure Log Rotation

Edit:

```bash
vi /etc/docker/daemon.json
```

Example:

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

Note:

```text
Log restrictions are usually more explicit for newly created containers
Existing packagings suggest recreate authentication
```

---

## XIII. Case 11: /var/lib/docker Occupies Too Much Space

---

## Phenomenon

Check disk:

```bash
df -h
```

Check Docker directory:

```bash
du -sh /var/lib/docker
```

Occupies a lot.

---

## Common Causes

```text
Too many mirrors.
Stop the container.
volume Too much.
Build too many caches
The container log is too big.
Uncleaned old version mirror
```

---

## Troubleshooting Steps

Check Docker usage:

```bash
docker system df
```

Check detailed usage:

```bash
docker system df -v
```

Check images:

```bash
docker images
```

Check all containers:

```bash
docker ps -a
```

Check volumes:

```bash
docker volume ls
```

Check Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

---

## Cleanup Commands

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

Clean unused resources:

```bash
docker system prune -a
```

---

## Note

Exercise caution in production environments:

```bash
docker volume prune
```

Reason:

```text
volume Possible database data, upload files, business continuity data
```

---

## XIV. Case 12: Volume Mounting Abnormality

---

## Phenomenon

Container starts normally, but application data is not saved as expected, or files on the host machine are not visible inside the container.

---

## Common Causes

```text
Host path does not exist
Host directory permissions are incorrect
volume Synchronising folder
Error inside container path
bind mount Covered original files in mirror
Could not close temporary folder: %s
SELinux / AppArmor Limits
```

---

## Troubleshooting Steps

Check container details:

```bash
docker inspect ContainersID
```

Check mount information:

```bash
docker inspect ContainersID | grep Mounts -A 30
```

Check volume:

```bash
docker volume ls
```

Check volume details:

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

## Handling Approach

```text
Confirm Mount Source Path
→ Confirm target path inside the container
→ Confirm Permissions
→ Confirm container running user
→ Make sure the mirror file is covered.
```

---

## XV. Case 13: Volume Data Accidental Deletion Risk

---

## High-risk Commands

```bash
docker volume prune
```

```bash
docker compose down -v
```

---

## Risk

These commands may delete data volumes not referenced by containers.

If the volume contains:

```text
MySQL Data
PostgreSQL Data
Redis Enduring Document
Apply Upload File
Operational configuration
```

Deletion may cause data loss.

---

## Troubleshooting and Verification

Check volume:

```bash
docker volume ls
```

Check details:

```bash
docker volume inspect volumeName
```

Check container mounts:

```bash
docker inspect ContainersID | grep Mounts -A 30
```

---

## Production Recommendations

```text
Clear volume Data value must be confirmed before doing so
Database volume There must be backup.
Do not execute when the scope of impact is unclear docker compose down -v
```

---

## XVI. Case 14: docker cp Path or Permission Issues

---

## Phenomenon

Execute:

```bash
docker cp ContainersID:/Path /Host Path
```

Failed.

---

## Common Causes

```text
Containers ID Or wrong name
The path inside the container does not exist
Host destination path does not exist
Insufficient Permissions
Path does not write absolute path
The container has been removed.
```

---

## Troubleshooting Steps

Check container:

```bash
docker ps -a
```

Confirm container path:

```bash
docker exec -it ContainersID /bin/sh
```

```bash
ls -lah /etc/nginx
```

Copy file:

```bash
docker cp nginx:/etc/nginx/nginx.conf /root/
```

Check host directory:

```bash
ls -ld /root/
```

Copy and check permissions:

```bash
ls -lh /root/nginx.conf
```

Modify permissions if necessary:

```bash
chown root:root /root/nginx.conf
```

```bash
chmod 644 /root/nginx.conf
```

---

## Note

Stopped containers can also be copied as long as the container object still exists.

Check stopped container:

```bash
docker ps -a
```

---

## 17I don't know.Case 15: Docker Compose Service Startup Failure

---

## Phenomenon

Execute:

```bash
docker compose up -d
```

Some services fail to start or exit.

---

## Common Causes

```text
YAML Syntax Error
Indentation Error
Mirror pull failed
Port Conflict
volume Mount error
Environmental variables are missing
Service start command error
Reliance on services not available
```

---

## Troubleshooting Steps

Check configuration:

```bash
docker compose config
```

Check status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f
```

Check specific service logs:

```bash
docker compose logs -f Service Name
```

Enter service container:

```bash
docker compose exec Service Name /bin/sh
```

---

## Port Conflict Troubleshooting

Check host listening:

```bash
ss -tunlp
```

If Compose has:

```yaml
ports:
  - "8080:80"
```

But host port 8080 is occupied, need to change port:

```yaml
ports:
  - "8081:80"
```

Restart:

```bash
docker compose up -d
```

---

## 18I don't know.Case 16: depends_on configured but service still connection failure

---

## Phenomenon

Compose file has already configured:

```yaml
depends_on:
  - mysql
```

But web service still reports database connection failure when starting.

---

## Cause

```text
depends_on Default Primary Control Start Order
It doesn't mean MySQL Initialization completed
Doesn't mean the database is ready for connection
```

---

## Recommended Approach

Use:

```text
depends_on + healthcheck + Apply its own retest mechanism
```

Example:

```yaml
services:
  web:
    image: my-web:v1
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Check status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f mysql
```

---

## 19I don't know.Case 17: healthcheck always unhealthy

---

## Phenomenon

Check:

```bash
docker compose ps
```

Status shows:

```text
unhealthy
```

---

## Common Causes

```text
There's no health check order.
Not in the mirror. curl / mysqladmin / redis-cli
Health check port error
It's a long start.start_period Too short.
Authentication parameter error
Health interface returned 2xx
localhost Pointing at not meeting expectations.
```

---

## Troubleshooting Steps

Check status:

```bash
docker compose ps
```

Check logs:

§

Check inspect:

```bash
docker inspect ContainersID
```

Focus on:

```text
State.Health.Status
State.Health.Log
```

Manually execute health check command in container:

```bash
docker compose exec Service Name /bin/sh
```

For example:

```bash
curl -f http://localhost:8080/health
```

---

## Handling Approach

```text
Confirm health check-ups.
→ Confirm port and path correct
→ Confirm that authentication parameters are correct
→ Bigger start_period
→ View Apply Real Log
```

---

## 20I don't know.Case 18: Harbor HTTP/HTTPS error

---

## Phenomenon

Docker pull or push error:

```text
http: server gave HTTP response to HTTPS client
```

---

## Cause

```text
Client Default Press HTTPS Visit repository
But... Harbor Actually... HTTP
```

---

## Docker Handling

Edit:

```bash
vi /etc/docker/daemon.json
```

Configuration:

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

## containerd Handling

Create directory:

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

Check config_path:

```bash
grep -n "config_path" /etc/containerd/config.toml
```

Should include:

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

Restart containerd:

```bash
systemctl restart containerd
```

---

## 21I don't know.Case 19: docker login succeeds but K8s pulls image failure

---

## Phenomenon

Execute on node:

```bash
docker login 10.0.0.10:8090
```

Succeeds.

But Kubernetes Pod fails to pull image.

---

## Core Cause

```text
K8s Use Node containerd Time
Pod Pull the mirror. containerd
Not going. Docker Login Status
```

Therefore:

```text
docker login Success
≠ K8s Pod I'm sure it'll pull a private mirror.
```

---

## Troubleshooting Steps

Check Pod events:

```bash
kubectl describe pod PodName -n Namespace
```

Check containerd configuration:

```bash
cat /etc/containerd/config.toml
```

Check Harbor hosts.toml:

```bash
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml
```

Test with crictl:

```bash
crictl pull 10.0.0.10:8090/project/nginx:latest
```

Test with ctr:

```bash
ctr -n k8s.io images pull 10.0.0.10:8090/project/nginx:latest
```

Check Secret:

```bash
kubectl get secret -n Namespace
```

Check Pod YAML:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

---

## Correct Approach

Create imagePullSecret:

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=10.0.0.10:8090 \
  --docker-username=admin \
  --docker-password='Harbor12345' \
  --docker-email=admin@example.com \
  -n default
```

Pod reference:

```yaml
imagePullSecrets:
  - name: harbor-secret
```

Note:

```text
Secret Must and Pod In the same one. namespace
```

---

## 22I don't know.Case 20: unauthorized: authentication required

---

## Phenomenon

Error when pulling/pushing image:

```text
unauthorized: authentication required
```

---

## Common Causes

```text
Not Login Harbor
Account password error
Account No Permissions
Harbor The project is private.
Mirror Path Error
K8s Not configured imagePullSecrets
Secret namespace Wrong.
```

---

## Docker Troubleshooting

Login:

```bash
docker login 10.0.0.10:8090
```

Pull:

```bash
docker pull 10.0.0.10:8090/project/nginx:latest
```

Check authentication file:

```bash
cat ~/.docker/config.json
```

Note:

```text
~/.docker/config.json It may contain sensitive information and not just disclose it.
```

---

## Kubernetes Troubleshooting

Check Secret:

```bash
kubectl get secret -n Namespace
```

Check Pod:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

Check events:

```bash
kubectl describe pod PodName -n Namespace
```

---

## 23I don't know.Case 21: Image push rejected

---

## Phenomenon

Execute:

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

Failed.

---

## Common Causes

```text
Not Login Harbor
No account push Permissions
Project does not exist
Mirror tag Path does not match Harbor Project Path
Harbor Read-only items
The warehouse is full.
```

---

## Troubleshooting Steps

Check local image:

```bash
docker images
```

Re-tag:

```bash
docker tag nginx:latest 10.0.0.10:8090/project/nginx:latest
```

Login to Harbor:

```bash
docker login 10.0.0.10:8090
```

Push:

```bash
docker push 10.0.0.10:8090/project/nginx:latest
```

---

## Correct tag format

```text
HarborAddress/Project name/Mirror Name:Label
```

Example:

```text
10.0.0.10:8090/project/nginx:latest
```

---

## 24I don't know.Case 22: containerd namespace misread causing image "not found"

---

## Phenomenon

Kubernetes Pod is running, but executing:

```bash
ctr images ls
```

Cannot see image.

---

## Cause

containerd has namespace concept.

When Kubernetes uses containerd, focus is on:

```text
k8s.io
```

Not the default namespace.

---

## Correct Check

```bash
ctr -n k8s.io images ls
```

Or:

```bash
nerdctl --namespace k8s.io images ls
```

Check crictl image:

```bash
crictl images
```

---

## Understanding

```text
ctr images ls
→ Maybe it was. default namespace

ctr -n k8s.io images ls
→ View Kubernetes Use namespace
```

---

## 25I don't know.Case 23: Container resource usage too high

---

## Phenomenon

Host CPU or memory usage is high, suspecting abnormal container.

---

## Troubleshooting Steps

Check container resources:

```bash
docker stats
```

Check container:

```bash
docker ps
```

Check logs:

```bash
docker logs --tail 100 ContainersID
```

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Check processes inside container:

```bash
top
```

Or:

```bash
ps aux
```

---

## Common Causes

```text
Apply Dead Cycle
Request traffic abnormal.
Memory Leak
Too many log outputs
No resource limitations set
Overoccupied single container CPU
Multiple containers fighting for resources
```

---

## Handling Recommendations

Limit resources when running container:

```bash
docker run -d --cpus="2" -m 1g nginx
```

Check resources:

```bash
docker stats
```

---

## Twenty-sixth, Case 24: Container OOM or Memory Insufficient

---

## Phenomenon

Container abnormally exits, suspected OOM.

---

## Troubleshooting Steps

Check container status:

```bash
docker ps -a
```

Check logs:

```bash
docker logs ContainersID
```

Check inspect:

```bash
docker inspect ContainersID
```

Focus on:

```text
State.OOMKilled
State.ExitCode
HostConfig.Memory
```

Can use:

```bash
docker inspect ContainersID | grep -i oom -A 5
```

Check host kernel logs:

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
Container memory limit is too small
Apply memory leaks
There's a lot of traffic.
JVM / Node.js Unreasonable memory parameters for running
Host's overall memory is insufficient
```

---

## Handling Approach

```text
Confirm if OOMKilled
→ View Application Log
→ View container memory limits
→ Adjust applied memory parameters
→ Adjust Containers memory limit
→ Capacity assessment
```

---

## Twenty-seventh, Case 25: Insufficient Permissions Inside Container

---

## Phenomenon

Command execution inside container fails:

```text
Operation not permitted
```

Or:

```text
Permission denied
```

---

## Common Causes

```text
Containers Not root User Run
Inadequate document permissions
Missing Linux capability
Device not map
seccomp / AppArmor / SELinux Limits
Nothing. privileged Permissions
```

---

## Troubleshooting Steps

Check container user:

```bash
docker exec -it ContainersID id
```

Check file permissions:

```bash
docker exec -it ContainersID ls -lah /Path
```

Check container configuration:

```bash
docker inspect ContainersID
```

---

## Common Capability Scenarios

Need network management capability:

```bash
docker run -d --cap-add NET_ADMIN nginx
```

Need raw packet capability:

```bash
docker run -d --cap-add NET_RAW nginx
```

Need process debugging capability:

```bash
docker run -d --cap-add SYS_PTRACE nginx
```

Device mapping:

```bash
docker run -d --device /dev/sdb:/dev/sdb nginx
```

Privileged mode:

```bash
docker run -d --privileged nginx
```

---

## Production Recommendations

```text
Don't go straight to the point of access. privileged
Let's judge what is missing.
Priority cap-add
Use of equipment when required --device
Last thought. --privileged
```

---

## Twenty-eighth, Case 26: Container Cannot Ping or Has No Permissions

---

## Phenomenon

Execute inside container:

```bash
ping 8.8.8.8
```

Error:

```text
Operation not permitted
```

---

## Possible Causes

```text
Missing NET_RAW
Not in the mirror. ping Command
The network itself doesn't work.
Security strategy limits
```

---

## Troubleshooting Steps

Enter container:

```bash
docker exec -it ContainersID /bin/sh
```

Check if ping exists:

```bash
which ping
```

Check routing:

```bash
ip route
```

Temporary test:

```bash
docker run --rm -it --cap-add NET_RAW busybox /bin/sh
```

Test after entering:

```bash
ping 8.8.8.8
```

---

## Note

```text
ping It doesn't necessarily mean business. TCP It's not working.
Some environments are banned. ICMPbut TCP Access normal
```

Can use:

```bash
curl -I http://example.com
```

Or:

```bash
nc -vz ObjectiveIP Port
```

---

## Twenty-ninth, Case 27: Container Cannot Access Host Services

---

## Phenomenon

Service is listening on host:

```text
127.0.0.1:3306
```

Container access fails.

---

## Cause

Inside container:

```text
127.0.0.1
```

Refers to the container itself, not the host.

---

## Troubleshooting Steps

Check listening on host:

```bash
ss -tunlp
```

Check Docker bridge gateway:

```bash
ip addr show docker0
```

Check routing inside container:

```bash
ip route
```

---

## Common Access Methods

Use host docker0 address:

```text
172.17.0.1
```

Access from container:

```bash
curl http://172.17.0.1:Port
```

If service only listens on 127.0.0.1, container may not be able to access.

Need service to listen:

```text
0.0.0.0
```

Or host internal IP.

---

## Thirtieth, Case 28: Time Inconsistency Between Container and Host

---

## Phenomenon

Time inside container is inconsistent with host, log time is incorrect.

---

## Common Causes

```text
Container time zone not configured
Timezone file missing in mirror
Apply your own time zone configuration error
Log system press UTC Output
```

---

## Troubleshooting Steps

Check time on host:

```bash
date
```

Check time inside container:

```bash
docker exec -it ContainersID date
```

Check timezone:

```bash
docker exec -it ContainersID cat /etc/timezone
```

---

## Handling Methods

Set TZ when running:

```bash
docker run -d -e TZ=Asia/Shanghai nginx
```

Set in Dockerfile:

```dockerfile
ENV TZ=Asia/Shanghai
```

Set in Compose:

```yaml
services:
  web:
    image: nginx:1.27
    environment:
      TZ: Asia/Shanghai
```

---

## Thirty-first, Case 29: Docker Service Itself Abnormal

---

## Phenomenon

Docker command execution fails:

```bash
docker ps
```

Error.

---

## Common Causes

```text
Docker Service not started
Docker socket Permission Abnormal
Disk Full
Docker Configure Error
daemon.json Format error
containerd Unusual
```

---

## Troubleshooting Steps

Check Docker status:

```bash
systemctl status docker
```

Check Docker logs:

```bash
journalctl -u docker -f
```

Check configuration:

```bash
cat /etc/docker/daemon.json
```

Verify JSON format:

```bash
python3 -m json.tool /etc/docker/daemon.json
```

Restart Docker:

```bash
systemctl restart docker
```

Check containerd:

```bash
systemctl status containerd
```

---

## Note

```text
Production environment restarted Docker Could affect the container in operation
Need to identify business impact and maintenance windows
```

---

## Thirty-second, Case 30: containerd Service Abnormal

---

## Phenomenon

Kubernetes node Pod abnormal, crictl or ctr commands abnormal.

---

## Troubleshooting Steps

Check containerd status:

```bash
systemctl status containerd
```

Check logs:

```bash
journalctl -u containerd -f
```

Check configuration:

```bash
cat /etc/containerd/config.toml
```

Check kubelet:

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet -f
```

Check crictl:

```bash
crictl ps -a
```

```bash
crictl images
```

---

## Common Causes

```text
containerd Configure Error
hosts.toml Configure Error
registry config_path Error
sandbox image Pull failed
Disk space is insufficient
containerd Data catalog anomaly
kubelet and containerd Communications anomaly
```

---

## Thirty-third, Production Troubleshooting General Command List

---

## Docker Basics

```bash
docker version
```

```bash
docker info
```

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker images
```

```bash
docker logs --tail 100 -f ContainersID
```

```bash
docker inspect ContainersID
```

```bash
docker stats
```

```bash
docker system df
```

---

## Network Troubleshooting

```bash
ss -tunlp
```

```bash
ip route
```

```bash
iptables -L -n
```

```bash
iptables -t nat -L -n
```

```bash
docker network ls
```

```bash
docker network inspect bridge
```

```bash
docker exec -it ContainersID ip route
```

```bash
docker exec -it ContainersID cat /etc/resolv.conf
```

---

## Storage and Log Troubleshooting

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
docker compose logs -f Service Name
```

```bash
docker compose exec Service Name /bin/sh
```

```bash
docker compose down
```

---

## Harbor / Image Troubleshooting

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

## containerd / K8s Node Troubleshooting

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
kubectl describe pod PodName -n Namespace
```

```bash
journalctl -u kubelet -f
```

---

## 34. Troubleshooting Priority Recommendations

Production troubleshooting priority recommendations:

```text
1. First confirm the extent of the impact.
2. Keep on-site information
3. Check logs and status again
4. Don't delete the container. volumeDemolitions
5. Don't start blindly. Docker
6. Don't be direct. docker system prune -a
7. Don't be direct. docker compose down -v
8. Don't come up. privileged
9. We need to locate the cause, then we need to do the repairs.
10. Validation and duplication after repair
```

---

## 35. One-Sentence Summary

Docker Production Troubleshooting Core is:

```text
Look at the state.
→ Read the log.
→ Look at the configuration
→ Look at the network.
→ Look at the memory.
→ Watch permissions.
→ Look at the resources.
→ Look at the mirror warehouse.
→ Look at the running boundary
```

Basic troubleshooting chain:

```text
docker ps -a
→ docker logs
→ docker inspect
→ docker stats
→ docker system df
```

Network troubleshooting chain:

```text
docker port
→ ss -tunlp
→ ip route
→ iptables
→ docker network inspect
→ Inside the container DNS / Route testing
```

Storage troubleshooting chain:

```text
df -h
→ docker system df
→ du -sh /var/lib/docker
→ View container log file
→ View volume and Mounts
```

Image pull troubleshooting chain:

```text
docker pull
→ docker login
→ Harbor Items and authority
→ HTTP / HTTPS Configure
→ containerd hosts.toml
→ crictl / ctr Test
→ K8s imagePullSecrets
```

Compose troubleshooting chain:

§

Production principles:

```text
Watch and then act.
Back up, then clean up.
Position first, then restart.
Minimum permission first, then privileged
Validation of data before deletion volume
Make sure you run it first, then you judge. Docker / containerd / K8s Border
``` /think