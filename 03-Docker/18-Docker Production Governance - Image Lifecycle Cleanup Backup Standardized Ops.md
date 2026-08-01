# 18-Docker Production Governance: Image Lifecycle, Cleanup Policies, Backup and Standardized Operations

#Docker #ProductionGovernance #MirrorLifeCycle #CleanupPolicy #BackupRestore #StandardizedTransport #Harbor #ContainerGovernance #Inspection #Transport

---

## Recommended Path

03-Container Technology/18-Docker Production Governance: Image Lifecycle, Cleanup Policies, Backup and Standardized Operations.md

---

## I. Document Explanation

This document organizes long-term governance content for Docker production environments, focusing on:

- Why Docker needs production governance
- Scope of Docker production governance
- Image lifecycle management
- Image tag standards
- Image retention policies
- Harbor project governance
- Image cleanup policies
- Container cleanup policies
- Volume cleanup risks
- Docker log governance
- Docker data directory planning
- Docker configuration backup
- Image backup and offline migration
- Volume data backup
- Compose project backup
- Docker host standardization baseline
- Docker inspection checklist
- Docker change standards
- Docker incident review and governance closure

The goal is:

- To establish Docker production governance thinking
→ To standardize image tag and image lifecycle
→ To safely clean up images, containers, logs, and caches
→ To distinguish which resources can be cleaned and which cannot be deleted arbitrarily
→ To backup Docker configuration, images, data volumes, and Compose projects
→ To form a standardized operation baseline for Docker hosts

---

## II. Why Docker Needs Production Governance

Docker is easy to use:

```bash
docker run -d nginx
```

But after long-term operation in production environments, common issues will increase:

```text
More and more mirrors.
The old version is empty.
latest Over and over again.
Harbor Project chaos
The container log is full of disks.
Stop container long-term residues
Useless volume No confirmation.
Docker Data directory full of system disks
Production containers without resource constraints
Container Default root Run
privileged Abused
Docker Configure No Backup
I can't recover from a malfunction.
```

Therefore, Docker production governance is not "knowing commands", but to solve:

```text
Stable long-term operation
Availability of resources
Is the mirror traceable?
Recoverability of data
Minimize Permissions
Is the failure fast-tracked?
```

One-sentence understanding:

```text
Docker Production governance = Jean. Docker Environmental maintenance, traceability, clean-up, recoverable, auditable
```

---

## III. Scope of Docker Production Governance

Docker production governance mainly includes:

```text
Mirror Governance
Container governance
Network governance
Storage governance
Log governance
Security governance
Configure governance
Backup Restore
Inspection surveillance
Change of norms
Fault Rewind
```

Corresponding relationships:

```text
Mirror Governance
→ tag Norms, reservations strategies, gap scanning,Harbor Clear

Container governance
→ Naming norms, resource limitations, restart strategy, running permissions

Storage governance
→ data-rootI don't know.volumeI don't know.bind mountDisk Space

Log governance
→ json-file Restrictions, log rotation, collection systems

Security governance
→ Not rootI don't know.capabilityI don't know.seccompI don't know.AppArmorMinimum permissions

Configure governance
→ daemon.jsonI don't know.hosts.tomlI don't know.Compose File, running parameters

Backup Restore
→ Mirrors,volumeI don't know.Compose Projects,Docker Configure

Inspection surveillance
→ Resource occupation, disks, logs, abnormal containers, mirror accumulation

Change of norms
→ Modify configuration, restart Docker, clean-up resources, upgrades
```

---

## IV. Image Lifecycle Management

---

## Scenario 1: What is an Image Lifecycle

A production image typically goes through:

```text
Develop Build
→ Local Test
→ Clear scan.
→ Send Harbor
→ Test environmental deployment
→ Advance release authentication
→ Production release
→ Roll Back
→ History Archive
→ Expired cleanup
```

Without lifecycle management, the following will occur:

```text
I don't know which mirror is being used.
I don't know which mirror can be removed.
I don't know which mirror corresponds to which code version.
I don't know what to use to roll back. tag
Harbor Storage continues to expand
```

---

## Scenario 2: Why Image Tags Are Important

It is not recommended to use only:

```text
latest
```

Reasons:

```text
latest It changes.
Unable to accurately track version
It's not clear.
I don't know the code at the checkup.
What a mess.
CI/CD Uncontrollable
```

Recommended tags should include:

```text
Apply Name
Environment
Branch Name
commitID
pipelineID
Version Number
Build Time
```

Example:

```text
myapp:main-a1b2c3d-1024
myapp:release-1.2.0-a1b2c3d
myapp:prod-20260425-a1b2c3d
```

---

## Scenario 3: Recommended Image Tag Standards

Basic format:

```text
Apply Name:Branch Name-commitID-pipelineID
```

Example:

```text
devatlas-api:main-a1b2c3d-1024
```

With environment:

```text
Apply Name:Environment-Version Number-commitID
```

Example:

```text
devatlas-api:prod-v1.2.0-a1b2c3d
```

With date:

```text
Apply Name:Environment-Date-commitID
```

Example:

```text
devatlas-api:prod-20260425-a1b2c3d
```

---

## Scenario 4: Build Trackable Tags in CI/CD

Example:

```bash
docker buildx build \
  --push \
  -t 10.0.0.10:8090/project/myapp:${CI_COMMIT_REF_NAME}-${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID} \
  .
```

Explanation:

```text
CI_COMMIT_REF_NAME
→ Branch Name

CI_COMMIT_SHORT_SHA
→ commit Short ID

CI_PIPELINE_ID
→ Waterline ID
```

Benefits:

```text
We can track the code.
We can track the water lines.
You can track the environment.
You can support the rollback.
```

---

## Scenario 5: Correct Usage of latest

`latest` It is not completely unusable, but not recommended as the sole production basis.

It can be used as an auxiliary tag:

```bash
docker tag myapp:main-a1b2c3d-1024 myapp:latest
```

Push:

```bash
docker push 10.0.0.10:8090/project/myapp:latest
```

But production deployment should prioritize explicit versions:

```text
myapp:prod-v1.2.0-a1b2c3d
```

Not recommended to use only:

```yaml
image: myapp:latest
```

More recommended:

```yaml
image: 10.0.0.10:8090/project/myapp:prod-v1.2.0-a1b2c3d
```

---

## V. Harbor Project Governance

---

## Scenario 6: Harbor Project Naming Standards

Projects should be divided by business or environment.

Example:

```text
base
middleware
dev
test
prod
platform
ops-tools
```

Projects can also be divided by team or system:

```text
devatlas
sre-platform
ops-tools
business-a
business-b
```

Not recommended:

```text
All the mirrors. library
All environments share the same mess.
Project name has no business implications
```

---

## Scenario 7: Harbor Permission Governance

Harbor projects should distinguish:

```text
Administrator
Developer
Maintainer
Read-only users
CI/CD Send the account number.
K8s Pull the account number.
```

Recommended:

```text
CI/CD Use a special account number push
Kubernetes Use specific robot account pull
No common user sharing admin
Production projects are not allowed anonymously.
Minimum Permission Authorization
Regular cleanup of useless accounts
```

---

## Scenario 8: Harbor Image Retention Policies

Recommended to set image retention policies, for example:

```text
Keep Recent 10 individual tag
Keep Recent 30 Mirror
Reservations prod / release Start tag
Clear dev / test Expiry tag
```

Example policy thinking:

```text
prod Item
→ Keep Recent 20 Production version
→ Keep All release Version for a while
→ Do not automatically clean the current production version

test Item
→ Keep Recent 10 Versions
→ Clear 30 Mirror of the sky

dev Item
→ Keep Recent 5 Versions
→ Clear 7 Mirror of the sky
```

---

## Scenario 9: Harbor Garbage Collection

After deleting tags in Harbor, the underlying storage space may not be released immediately.

Usually, you also need to execute:

```text
Garbage Collection
```

Governance thinking:

```text
Set retention policy first
→ Delete Expiry tag
→ Implementation Garbage Collection
→ Observation Harbor Storage space
```

Note:

```text
Harbor Garbage recovery should arrange maintenance windows
Avoiding influence. push / pull Tasks
```

---

## VI. Image Cleanup Policies

---

## Scenario 10: View Local Images

```bash
docker images
```

Check image usage:

```bash
docker system df
```

Check detailed usage:

```bash
docker system df -v
```

---

## Scenario 11: Clean Dangling Images

Dangling images are usually displayed as:

```text
<none>:<none>
```

Cleanup:

```bash
docker image prune
```

Force cleanup:

```bash
docker image prune -f
```

Explanation:

```text
dangling The mirror is usually built or hit again. tag Unlabeled mirror behind
```

---

## Scenario 12: Clean Unused Images

Clean images not used by containers:

```bash
docker image prune -a
```

Note:

```text
This command will remove all currently unquoted mirrors.
Even if these mirrors could be used to roll back.
```

Use with caution in production environments.

A more secure approach:

```text
First docker images View
What else? tag Not anymore.
And press the mirror. ID or tag Delete
```

Delete specific images:

```bash
docker rmi MirrorID
```

Or:

```bash
docker rmi Mirror Name:Label
```

---

## Scenario 13: Clean Images by Time

Clean images not used in 30 days:

```bash
docker image prune -a --filter "until=720h"
```

Explanation:

```text
720h = 30 days
```

Clean images not used in 7 days:

```bash
docker image prune -a --filter "until=168h"
```

Production recommendation:

```text
First test the environmental clearance strategy.
Retain current and rolling versions before cleaning the production environment
```

---

## Scenario 14: Pre-Cleanup Checks

Before cleanup, it is recommended to execute:

```bash
docker ps -a
```

```bash
docker images
```

```bash
docker system df -v
```

Confirm:

```text
Which mirrors are currently used for the running container
Is there a mirror that needs to be saved to roll back?
Is there a production mirror?
Is there a temporary test mirror?
```

---

## VII. Container Cleanup Policies

---

## Scenario 15: View Stopped Containers

```bash
docker ps -a
```

Filter stopped containers:

```bash
docker ps -a --filter "status=exited"
```

---

## Scenario 16: Clean Stopped Containers

Clean all stopped containers:

```bash
docker container prune
```

Force cleanup:

```bash
docker container prune -f
```

Explanation:

```text
Stopping containers usually clean up.
But we need to confirm whether the site needs to be preserved before we clean up.
```

---

## Scenario 17: Do Not Rush to Delete Containers During Production Troubleshooting

If a container exits abnormally, do not delete it immediately.

First, preserve the scene:

```bash
docker ps -a
```

```bash
docker logs ContainersID
```

```bash
docker inspect ContainersID
```

If necessary, copy logs or configurations:

```bash
docker cp ContainersID:/app/logs /tmp/app-logs
```

Delete only after confirming no preservation value:

```bash
docker rm ContainersID
```

---

## Scenario 18: Clean Containers by Conditions

Clean containers stopped 7 days ago:

```bash
docker container prune --filter "until=168h"
```

Clean specific containers:

```bash
docker rm ContainersID
```

Force deletion:

```bash
docker rm -f ContainersID
```

Note:

```text
docker rm -f Will stop and remove the container.
Pre-production recognition of impact ranges
```

---

## VIII. Volume Cleanup Policies and Risks

---

## Scenario 19: View Volumes

```bash
docker volume ls
```

Check details:

```bash
docker volume inspect volumeName
```

---

## Scenario 20: Why Volumes Cannot Be Deleted Arbitrarily

Volumes may contain: /think

```text
MySQL Data
PostgreSQL Data
Redis Durable data
Apply Upload File
Operational configuration
Intermediate status data
```

High-risk commands:

```bash
docker volume prune
```

```bash
docker compose down -v
```

Risk:

```text
Delete those not quoted in the container volume
Possible loss of databases or operational data
```

---

## Scenario 21: Confirm Before Volume Cleanup

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect volumeName
```

Check which containers are mounted to the volume:

```bash
docker ps -a --filter volume=volumeName
```

Check container mounts:

```bash
docker inspect ContainersID | grep -A 30 Mounts
```

---

## Scenario 22: Delete Specified Volume

Delete after confirming it's no longer needed:

```bash
docker volume rm volumeName
```

If deletion fails, it may still be referenced by containers.

Check references:

```bash
docker ps -a --filter volume=volumeName
```

---

## Scenario 23: Production Volume Cleanup Principles

```text
I don't know what to use. volume No, no, no.
Database-related volume Do not just delete
Backup before cleaning
Confirm container reference relationship before cleanup
Prior to clearance, identify the chief of operations
No difference between scripts docker volume prune
```

---

## IX. Docker Log Governance

---

## Scenario 24: Risks of Default json-file Logging Driver

Docker's default common logging driver:

```text
json-file
```

Container standard output writes to the host:

```bash
/var/lib/docker/containers/*/*.log
```

If not restricted, may result in:

```text
Dozens of individual container logs GB
/var/lib/docker Filled up.
The system is full.
Docker Or business anomaly.
```

---

## Scenario 25: Check Container Log Files

```bash
ls -lh /var/lib/docker/containers/*/*.log
```

Find large logs:

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

Check Docker usage:

```bash
docker system df
```

Check disk usage:

```bash
df -h
```

---

## Scenario 26: Configure Docker Log Rotation

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

Description:

```text
max-size
→ Maximum size of single log file

max-file
→ Keep up to a few log files
```

Note:

```text
The configuration is usually more explicit for newly created containers.
Existing packagings suggest recreate authentication
```

---

## Scenario 27: Container Log Governance Recommendations

```text
The production environment must have a log rotation.
Apply do not crazy output debug Log
The log should be collected. ELK / Loki / Log Platform
Local packaging logs are only kept for short periods
Important logs should not rely solely on local packaging files
```

---

## X. Docker Data Directory Planning

---

## Scenario 28: Check Docker Root Directory

```bash
docker info | grep "Docker Root Dir"
```

Common default paths:

```bash
/var/lib/docker
```

---

## Scenario 29: Why Not to Place Docker Data on System Disk Long-Term

If Docker data directory is on system disk:

```text
Mirror Growth
Container layer growth
volume Growth
Log Growth
Build Cache Growth
```

May lead to:

```text
System Full
Docker Unusual
The container cannot be activated.
The mirror cannot be pulled
System service anomaly
```

---

## Scenario 30: Modify Docker data-root

Edit:

```bash
vi /etc/docker/daemon.json
```

Example:

```json
{
  "data-root": "/data/docker"
}
```

Migration process:

```bash
systemctl stop docker
```

```bash
mkdir -p /data/docker
```

```bash
rsync -aP /var/lib/docker/ /data/docker/
```

```bash
vi /etc/docker/daemon.json
```

```bash
systemctl start docker
```

Verify:

```bash
docker info | grep "Docker Root Dir"
```

Production notes:

```text
Backup configuration before moving
Stop while moving Docker
Confirm that the disk space is adequate
Validation of operations before processing old directories
```

---

## XI. Docker Configuration Backup

---

## Scenario 31: What Docker Configurations to Backup

Common configurations:

```text
/etc/docker/daemon.json
/etc/containerd/config.toml
/etc/containerd/certs.d/
/etc/systemd/system/docker.service.d/
Compose Project Directory
Harbor hosts.toml
TLS Certificate
Private repository authentication configuration
```

---

## Scenario 32: Backup Docker Configurations

Create backup directory:

```bash
mkdir -p /backup/docker-config-$(date +%F)
```

Backup Docker configurations:

```bash
cp -a /etc/docker /backup/docker-config-$(date +%F)/
```

Backup containerd configurations:

```bash
cp -a /etc/containerd /backup/docker-config-$(date +%F)/
```

Backup systemd drop-in:

```bash
cp -a /etc/systemd/system/docker.service.d /backup/docker-config-$(date +%F)/ 2>/dev/null || true
```

Check backup:

```bash
ls -lah /backup/docker-config-$(date +%F)
```

---

## Scenario 33: Record Docker Information Before Backup

```bash
docker version > /backup/docker-config-$(date +%F)/docker-version.txt
```

```bash
docker info > /backup/docker-config-$(date +%F)/docker-info.txt
```

```bash
docker network ls > /backup/docker-config-$(date +%F)/docker-network-ls.txt
```

```bash
docker volume ls > /backup/docker-config-$(date +%F)/docker-volume-ls.txt
```

```bash
docker images > /backup/docker-config-$(date +%F)/docker-images.txt
```

```bash
docker ps -a > /backup/docker-config-$(date +%F)/docker-ps-a.txt
```

---

## XII. Image Backup and Offline Migration

---

## Scenario 34: Export Images

Export single image:

```bash
docker save -o myapp-v1.tar myapp:v1
```

Export multiple images:

```bash
docker save -o app-bundle.tar nginx:1.27 redis:7 mysql:8.0
```

Check files:

```bash
ls -lh *.tar
```

---

## Scenario 35: Import Images

```bash
docker load -i myapp-v1.tar
```

Verify:

```bash
docker images | grep myapp
```

---

## Scenario 36: Production Image Backup Recommendations

Suitable for backup:

```text
Key production versions
Could not reboot mirrors from the extranet
Mirror used in offline environment
Important intermediate mirror
Internal construction mirror
```

Not necessarily need to backup:

```text
Anytime. Harbor Pulling mirrors.
Temporary test mirror
No business value mirror
```

More recommended for long-term dependencies:

```text
Harbor
Mirror repository backup
Mirror tag Normative
CI/CD Rebuildable
```

Rather than relying solely on single-machine `docker save`.

---

## XIII. Volume Data Backup

---

## Scenario 37: Backup Named Volumes

Assume volume name is:

```text
mysql-data
```

Backup:

```bash
docker run --rm \
  -v mysql-data:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar czf /backup/mysql-data-$(date +%F).tar.gz -C /data .
```

Check backup:

```bash
ls -lh /backup/mysql-data-*.tar.gz
```

---

## Scenario 38: Restore Named Volumes

Create new volume:

```bash
docker volume create mysql-data-restore
```

Restore:

```bash
docker run --rm \
  -v mysql-data-restore:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar xzf /backup/mysql-data-2026-04-25.tar.gz -C /data
```

Check volume:

```bash
docker volume inspect mysql-data-restore
```

---

## Scenario 39: Database Backups Should Not Only Backup Volumes

For databases, directly backing up volumes carries risks.

More recommended is application-level backup:

```text
MySQL
→ mysqldump / xtrabackup

PostgreSQL
→ pg_dump / pg_basebackup

MongoDB
→ mongodump

Redis
→ RDB / AOF Backup
```

Volume backups are suitable for:

```text
Get ready.
Stop backup
Support Backup
File Data
Upload directory
Configure Directory
```

Database online backups should use database-specific tools.

---

## XIV. Compose Project Backup

---

## Scenario 40: What to Backup in Compose Projects

Compose projects recommend backing up:

```text
compose.yaml
compose.override.yaml
.env
env_file
Dockerfile
Start Script
Configure Directory
Mount directory
Database Backup
Relevant README
```

Not recommended to backup:

```text
node_modules
Temporary Log
Build Cache
Useless Test File
```

---

## Scenario 41: Backup Compose Project Directory

Assume project directory:

```bash
/opt/compose/myapp
```

Backup:

```bash
tar czf /backup/myapp-compose-$(date +%F).tar.gz -C /opt/compose myapp
```

Check:

```bash
ls -lh /backup/myapp-compose-*.tar.gz
```

---

## Scenario 42: Restore Compose Project

Unzip:

```bash
mkdir -p /opt/compose
```

```bash
tar xzf /backup/myapp-compose-2026-04-25.tar.gz -C /opt/compose
```

Enter directory:

```bash
cd /opt/compose/myapp
```

Check configuration:

```bash
docker compose config
```

Start:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f
```

---

## XV. Docker Host Standardization Baseline

---

## Scenario 43: Docker Host Directory Planning

Recommendations:

```text
System Disk
→ Operating systems,Docker Procedures

Data Disk
→ Docker data-root, container data, mirror cache

Backup disk or remote storage
→ Configure backup, database backup,volume Backup
```

Example:

```text
/data/docker
/data/compose
/data/logs
/backup
```

---

## Scenario 44: Docker daemon.json Baseline Example

Example:

```json
{
  "data-root": "/data/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "insecure-registries": ["10.0.0.10:8090"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

Description:

```text
data-root
→ Docker Data Directory

log-opts
→ Log Round

insecure-registries
→ Intranet HTTP Harbor

cgroupdriver
→ Kubernetes It's common. systemd
```

Note:

```text
Priority use in the production environment HTTPS Harbor
HTTP Harbor Only for controlled intranet environment
```

---

## Scenario 45: Backup Before Modifying Docker Configuration

```bash
cp -a /etc/docker/daemon.json /etc/docker/daemon.json.$(date +%F-%H%M%S).bak
```

Check JSON format:

```bash
python3 -m json.tool /etc/docker/daemon.json
```

Reload:

```bash
systemctl daemon-reload
```

Restart Docker:

```bash
systemctl restart docker
```

Check status:

```bash
systemctl status docker
```

---

## XVI. Docker Daily Inspection Checklist

---

## Scenario 46: Basic Status Inspection

Check Docker service:

```bash
systemctl status docker
```

Check Docker version:

```bash
docker version
```

Check Docker info:

```bash
docker info
```

Check containers:

```bash
docker ps
```

Check abnormal containers:

```bash
docker ps -a --filter "status=exited"
```

---

## Scenario 47: Resource Inspection

Check host disk usage:

```bash
df -h
```

Check Docker Usage:

```bash
docker system df
```

Check Detailed Usage:

```bash
docker system df -v
```

Check Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

Check Container Resources:

```bash
docker stats
```

---

## Scenario 48: Log Inspection

Check Large Logs:

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

Check Log Configuration:

```bash
docker info | grep -i "Logging Driver"
```

Check daemon.json:

```bash
cat /etc/docker/daemon.json
```

---

## Scenario 49: Image Inspection

Check Images:

```bash
docker images
```

Check Dangling Images:

```bash
docker images -f "dangling=true"
```

Check Image Usage:

```bash
docker system df -v
```

---

## Scenario 50: Volume Inspection

Check Volumes:

```bash
docker volume ls
```

Check Volume Details:

```bash
docker volume inspect volumeName
```

Check Container Mounts:

```bash
docker inspect ContainersID | grep -A 30 Mounts
```

---

## Scenario 51: Network Inspection

Check Networks:

```bash
docker network ls
```

Check bridge:

```bash
docker network inspect bridge
```

Check Host Routing:

```bash
ip route
```

Check iptables NAT:

```bash
iptables -t nat -L -n -v
```

---

## Section 17: Docker Change Guidelines

---

## Scenario 52: Which Operations Are High-Risk Changes

High-risk operations include:

```text
Restart Docker
Modify daemon.json
Modify Docker data-root
Modify Docker Default segment
Clear Docker Resources
Delete volume
Implementation docker system prune -a
Implementation docker compose down -v
Upgrade Docker Version
Modify containerd Configure
Modify Harbor HTTP / HTTPS Configure
Clear iptables
```

---

## Scenario 53: Pre-Change Checks for Docker

Pre-change recommendations:

```text
Recognition of operational impact
Confirm maintenance window
Confirm current running container
Confirm mirror and volume
Check for backup.
Confirm Rollback
Notification of the person concerned
```

Commands:

```bash
docker ps -a
```

```bash
docker images
```

```bash
docker volume ls
```

```bash
docker network ls
```

```bash
docker info
```

Backup Configuration:

```bash
mkdir -p /backup/docker-change-$(date +%F-%H%M%S)
```

```bash
cp -a /etc/docker /backup/docker-change-$(date +%F-%H%M%S)/
```

```bash
cp -a /etc/containerd /backup/docker-change-$(date +%F-%H%M%S)/ 2>/dev/null || true
```

---

## Scenario 54: Post-Change Verification for Docker

Post-change verification:

```bash
systemctl status docker
```

```bash
docker info
```

```bash
docker ps
```

```bash
docker system df
```

```bash
docker network ls
```

```bash
docker volume ls
```

If involving containers:

```bash
docker logs --tail 100 ContainersID
```

If involving Compose:

```bash
docker compose ps
```

```bash
docker compose logs --tail 100
```

---

## Section 18: Docker Cleanup Script Principles

---

## Scenario 55: Do Not Write Dangerous Unconfirmed Scripts

High-risk script examples:

```bash
docker system prune -a -f
docker volume prune -f
docker compose down -v
```

Risks:

```text
Undifferentiated Delete Resource
Maybe delete the rollback mirror
Possible deletion of business data
It could destroy the scene.
```

---

## Scenario 56: Capabilities of Cleanup Scripts

Production cleanup scripts should have:

```text
Only clear objects
First dry-run Presentation
White list protected.
Time to filter
Log log
Do not clean automatically volume
Do not clear an active mirror
No clean-up of production reservations tag
Second confirmation of implementation
```

---

## Scenario 57: Safe Cleanup Example

Only clean up dangling images older than 30 days:

```bash
docker image prune -f --filter "until=720h"
```

Only clean up stopped containers older than 7 days:

```bash
docker container prune -f --filter "until=168h"
```

Clean up build cache:

```bash
docker builder prune -f --filter "until=168h"
```

Check usage before cleanup:

```bash
docker system df
```

Check usage after cleanup:

```bash
docker system df
```

---

## Section 19: Docker Backup and Recovery Drills

---

## Scenario 58: Why Perform Recovery Drills

Backup is not the goal; recovery is.

Common issues:

```text
There's backup, but I don't know how to recover.
Backup file corrupted
Backup incomplete
Only the backup. Compose File, no backup. volume
Only the backup. volume, no backup database logical data
I'm not giving you the right access when I get back.
Post-recovered mirror missing
Missing environment variable after recovery
```

Therefore, regular recovery drills are essential.

---

## Scenario 59: Basic Recovery Drill Process

```text
Prepare to test the machine.
→ Install Docker
→ Restore Docker Configure
→ Import mirror or connection Harbor
→ Restore Compose Item
→ Restore volume Or database backup
→ docker compose config
→ docker compose up -d
→ Certification services
→ Records issues
```

---

## Scenario 60: Recovery Verification Commands

Check services:

```bash
docker ps
```

Check Compose:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f
```

Verify ports:

```bash
ss -tunlp
```

Access services:

```bash
curl -I http://127.0.0.1:8080
```

Check data:

```bash
docker exec -it ContainersID /bin/sh
```

---

## Section 20: Fault Review and Governance Closure

---

## Scenario 61: Focus Points for Docker Fault Review

Fault review should not only write:

```text
Restore after restart
```

Should record:

```text
Fault Time
Scope of impact
Fault phenomena
Find Method
Direct causes
Root analysis
Process
Recovery Time
Loss of data
Is there a security alarm?
Whether preventive measures exist
Follow-up and restructuring missions
```

---

## Scenario 62: Common Docker Fault Governance Actions

Logs filling disk:

```text
Supplementary log-opts
Access Log Collection
Limit Application debug Log
Add Disk Alert
```

Image pull failure:

```text
Normative imagePullSecrets
Normative Harbor Permissions
Configure HTTP / HTTPS
Node Side crictl Authentication
```

Volume mistakenly deleted:

```text
Limits down -v
Add Backup
Clean Script Ban volume prune
Additional approvals
```

Container resource exhaustion:

```text
Increased resource constraints
Increase surveillance
Optimization of application
Capacity assessment
```

Privileged abuse:

```text
For cap-add / device
Establishment of a security baseline
Audit of high-authorized containers
```

---

## Section 21: Production Governance Checklist

---

## 1. Image Governance

```text
Is there any? tag Normative
Avoidance of productive use latest
Whether or not to support mirror rollback
Whether to clear expired mirrors
Enable Harbor Scan
Whether to set Harbor Retain Policy
Key mirror backup or recapability capability
```

---

## 2. Container Governance

```text
Do packagings have naming instructions?
Whether to set restart Policy
Whether to set CPU / Memory Limit
Whether or not to avoid privileged
Whether or not to avoid host network
Whether to use Africa root User
Limit capability
Is there a health check-up?
```

---

## 3. Storage Governance

```text
Docker data-root Whether on the data disc
volume Clarity of use
Database volume Is there a backup?
Prohibition of non-confirmation volume prune
bind mount Justiciability of competence
Is there any disk space for alarm?
```

---

## 4. Log Governance

```text
Configure log-driver
Configure max-size / max-file
Access to log collection system
Whether to avoid infinity growth of container logs
Is there a big log check?
```

---

## 5. Security Governance

```text
Whether or not to avoid Docker socket Exposure
Limit docker Group Users
Whether to avoid mount host root directory
Whether to use no-new-privileges
Whether to use default seccomp / AppArmor
Whether high risk is restricted capability
```

---

## 6. Backup Governance

```text
Whether to backup daemon.json
Whether to backup containerd Configure
Whether to backup Compose Item
Whether to backup volume / Database
Is there any resumption exercise?
Whether to record recovery steps
```

---

## 7. Change Governance

```text
Restart Docker Whether there is a maintenance window
Modify daemon.json Do you want backup first?
Identification of clean-up resources
Delete volume Approvals
Upgrade Docker Is there a rollback option?
Change the network to assess impact
```

---

## Section 22: Common Command Summary

---

## Basic Status

Check Docker service:

```bash
systemctl status docker
```

Check Docker version:

```bash
docker version
```

Check Docker info:

```bash
docker info
```

Check containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

---

## Image Governance

Check images:

```bash
docker images
```

Check dangling images:

```bash
docker images -f "dangling=true"
```

Check image usage:

```bash
docker system df -v
```

Clean up dangling images:

```bash
docker image prune
```

Clean up unused images older than 30 days:

```bash
docker image prune -a --filter "until=720h"
```

Delete specific image:

```bash
docker rmi Mirror Name:Label
```

---

## Container Governance

Check stopped containers:

```bash
docker ps -a --filter "status=exited"
```

Clean up stopped containers:

```bash
docker container prune
```

Clean up stopped containers older than 7 days:

```bash
docker container prune --filter "until=168h"
```

Delete specific container:

```bash
docker rm ContainersID
```

---

## Volume Governance

Check volumes:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect volumeName
```

Check containers referencing volume:

```bash
docker ps -a --filter volume=volumeName
```

Delete specific volume:

```bash
docker volume rm volumeName
```

High-risk cleanup:

```bash
docker volume prune
```

---

## Log Governance

Check container logs:

```bash
docker logs --tail 100 -f ContainersID
```

Check large logs:

```bash
find /var/lib/docker/containers -name "*.log" -size +500M -exec ls -lh {} \;
```

Check Docker log configuration:

```bash
docker info | grep -i "Logging Driver"
```

Check daemon.json:

```bash
cat /etc/docker/daemon.json
```

---

## Disk Governance

Check disk:

```bash
df -h
```

Check Docker Root Dir:

```bash
docker info | grep "Docker Root Dir"
```

Check Docker directory usage:

```bash
du -sh /var/lib/docker
```

Check Docker resource usage:

```bash
docker system df
```

Check detailed usage:

```bash
docker system df -v
```

---

## Configuration Backup

Create backup directory:

```bash
mkdir -p /backup/docker-config-$(date +%F)
```

Backup Docker configuration:

```bash
cp -a /etc/docker /backup/docker-config-$(date +%F)/
```

Backup containerd configuration:

```bash
cp -a /etc/containerd /backup/docker-config-$(date +%F)/ 2>/dev/null || true
```

Record Docker information:

```bash
docker info > /backup/docker-config-$(date +%F)/docker-info.txt
```

---

## Image Backup

Export image:

```bash
docker save -o myapp-v1.tar myapp:v1
```

Import image:

```bash
docker load -i myapp-v1.tar
```

Export multiple images:

```bash
docker save -o app-bundle.tar nginx:1.27 redis:7 mysql:8.0
```

---

## Volume Backup

Backup volume:

```bash
docker run --rm \
  -v mysql-data:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar czf /backup/mysql-data-$(date +%F).tar.gz -C /data .
```

Restore volume:

§


Restore volume:

```bash
docker volume create mysql-data-restore
```

```bash
docker run --rm \
  -v mysql-data-restore:/data \
  -v /backup:/backup \
  alpine:3.20 \
  tar xzf /backup/mysql-data-2026-04-25.tar.gz -C /data
```

---

## Compose Backup and Restore

Backup Compose project:

```bash
tar czf /backup/myapp-compose-$(date +%F).tar.gz -C /opt/compose myapp
```

Restore Compose project:

```bash
tar xzf /backup/myapp-compose-2026-04-25.tar.gz -C /opt/compose
```

Check configuration:

```bash
docker compose config
```

Start:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

---

## Clean Cache

Clean builder cache:

```bash
docker builder prune
```

Clean builder cache from 7 days ago:

```bash
docker builder prune --filter "until=168h"
```

Clean buildx cache:

```bash
docker buildx prune
```

Check buildx cache usage:

```bash
docker buildx du
```

---

## Twenty-Three, One-Sentence Summary

The core of Docker production governance isn't "cleaning up resources", but:

```text
The reservation may be retained
The clean-up can clean up.
The backup can be restored.
There are limits to this restriction.
The tracker can track.
```

Image governance core:

```text
Don't just use it. latest
tag To track it.
Harbor Retention strategy
The old mirror needs to be cleaned regularly.
The key version needs to roll back.
```

Cleanup governance core:

```text
Mirrors can be cleaned up according to strategy.
Stop the container and clean it up.
build cache Sweepable
volume No brainless clean-up.
Logs must limit size
```

Backup governance core:

```text
Docker Configure to Backup
Compose Item to backup
The key mirrors must be restored.
Database needs backup with database tools
volume Backup to resume exercise.
```

Standardization governance core:

```text
data-root Disk
daemon.json A uniform baseline exists
It's in the log. max-size / max-file
Containers have resource limitations
Minimize Permissions
High-risk operations with maintenance windows and rollback programmes
```

Production principles:

```text
Check first, then clean.
Backup, change.
Confirm, then delete.
Let's get back to practice. We'll trust the backup.
Standardize, automate.
```