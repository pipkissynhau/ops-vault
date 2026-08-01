# 01-GitLab Backup and Recovery: Full Code Repository Backup, Configuration Protection, and Cross-Machine Recovery

Recommended path: 08-CI-CD/02-CI-CD Platform Backup and Disaster Recovery/01-GitLab Backup and Recovery: Full Code Repository Backup, Configuration Protection, and Cross-Machine Recovery.md

Tags: #CI-CD #GitLab #BackupRestore #DisasterPreparedness #CodeRepository #Transport #SRE

---

## I. Document Overview

GitLab is the code asset center in enterprise CI/CD systems, typically hosting the following core content:

- Git code repositories
- Users, groups, projects, and permission relationships
- Issues, Merge Requests, Wiki
- CI/CD Pipeline data
- CI/CD Variables
- Artifacts
- LFS files
- Uploads attachments
- Package Registry data
- Container Registry data (varies by deployment method)
- GitLab system configuration
- GitLab key files

In production environments, GitLab backups cannot be understood as merely "backing up the Git repository directory". A truly reliable GitLab backup should simultaneously cover:

1. Application data backup
2. Configuration file backup
3. Key file backup
4. Certificate and SSH Host Key backup
5. Cross-machine recovery verification
6. Scheduled backup tasks
7. Backup fileAlien. storage
8. Recovery drills

This document focuses on Linux Package / Omnibus GitLab deployment methods, organizing GitLab's full backup, configuration protection, recovery process, and production considerations.

---

## II. Why GitLab Needs Backup

If GitLab fails or data is lost, it affects not only code repositories but also the entire R&D delivery chain.

Common risks include:

- Server disk damage
- Accidental deletion of GitLab instance
- Database corruption
- Accidental deletion of projects or branches
- Failed GitLab upgrade
- Loss of CI/CD variables
- Runner authentication failure
- Loss of certificates or key files
- Incomplete data during migration to new servers
- Data unavailability due to attacks or accidental operations

GitLab's position in the CI/CD system is as follows:

    Developers
       |
       v
    GitLab Code Repository
       |
       v
    Jenkins / GitLab CI
       |
       v
    Harbor Image Repository
       |
       v
    Kubernetes Cluster

If GitLab is lost, subsequent Jenkins, Harbor, and Kubernetes release chains will all be affected.

Therefore, GitLab backup is a top priority in CI/CD platform disaster recovery.

---

## III. GitLab Backup Objects

### 3.1 Application Data Backup Objects

GitLab application data backup typically includes:

- Database data
- Git repositories
- Wiki repositories
- LFS objects
- Uploads attachments
- CI/CD Artifacts
- Pages data
- Packages data
- Terraform State (varies by version and configuration)
- Container Registry data (varies by configuration and storage method)

Common backup commands:

    sudo gitlab-backup create

Older versions or some documentation may also show:

    sudo gitlab-rake gitlab:backup:create

In production, prioritize the official documentation-recommended command for the current GitLab version.

---

### 3.2 Configuration File Backup Objects

In Linux Package deployment, the core configuration directory is:

    /etc/gitlab

Key files include:

    /etc/gitlab/gitlab.rb
    /etc/gitlab/gitlab-secrets.json

Where:

- gitlab.rb: Main GitLab configuration file
- gitlab-secrets.json: Stores database encryption keys, CI/CD variable encryption keys, 2FA-related keys, etc., for sensitive content

If only application data is backed up without gitlab-secrets.json, recovery may result in:

- CI/CD Variables unable to decrypt
- 2FA users unable to log in normally
- Runner authentication anomalies
- Webhook, Token, sensitive field anomalies
- Some encrypted data unable to recover

Configuration backup command:

    sudo gitlab-ctl backup-etc

Default generation directory:

    /etc/gitlab/config_backup/

---

### 3.3 Certificates and SSH Host Key

If GitLab uses HTTPS, certificate files must also be backed up, for example:

    /etc/gitlab/ssl/

For complete machine migration, it's also recommended to back up SSH Host Key:

    /etc/ssh/

The reason is that Git clients typically record the server's SSH fingerprint. If the SSH Host Key changes after migration, clients may show warnings like:

    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

This isn't necessarily a failure, but it affects user experience and may trigger security alerts.

---

## IV. Pre-Backup Environment Check

### 4.1 Check GitLab Version

When recovering GitLab, the target instance must match the backup source's version and type as much as possible.

Check version:

    sudo gitlab-rake gitlab:env:info

Or:

    sudo gitlab-ctl status
    cat /opt/gitlab/version-manifest.txt | head

You can also check in the GitLab Web interface:

    Admin Area -> Overview -> Dashboard

Need to record:

    GitLab version
    CE or EE
    Deployment method
    Operating system version
    External access domain
    HTTPS enabled
    External PostgreSQL used
    External Redis used
    Object storage used
    Container Registry enabled

---

### 4.2 Check Backup Path

Default application backup path is usually:

    /var/opt/gitlab/backups

Check configuration:

    sudo grep -n "backup_path" /etc/gitlab/gitlab.rb

If not separately configured, it typically uses the default path.

You can also explicitly configure it in gitlab.rb:

    gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"

After modifying the configuration, execute:

    sudo gitlab-ctl reconfigure

---

## V. Manual GitLab Backup

### 5.1 Create Application Data Backup

Execute: /think

sudo gitlab-backup create

After the backup is complete, check:

    sudo ls -lh /var/opt/gitlab/backups

Common backup file formats are similar to:

    1710000000_2026_04_28_16.11.0_gitlab_backup.tar

Where:

    1710000000_2026_04_28_16.11.0

is the BACKUP identifier needed for restoration.

When restoring, you don't need to write the full filename, just write:

    BACKUP=1710000000_2026_04_28_16.11.0

---

### 5.2 Create Configuration File Backup

Execute:

    sudo gitlab-ctl backup-etc

Check the configuration backup:

    sudo ls -lh /etc/gitlab/config_backup/

Common filenames are similar to:

    gitlab_config_1710000000_2026_04_28.tar

Configuration backups are very important, especially:

    gitlab.rb
    gitlab-secrets.json

In production environments, application data backups and configuration key backups should be saved separately to avoid attackers getting both data and decryption keys simultaneously.

---

### 5.3 Backup Certificate Directory

If using GitLab's built-in Nginx HTTPS, certificates may be in:

    /etc/gitlab/ssl/

Backup:

    sudo tar -czf /backup/gitlab/gitlab-ssl-$(date +%F).tar.gz /etc/gitlab/ssl

If certificates are managed by external Nginx or load balancer, they need to be backed up separately at their respective locations.

---

### 5.4 Backup SSH Host Key

Execute:

    sudo tar -czf /backup/gitlab/ssh-host-keys-$(date +%F).tar.gz /etc/ssh/ssh_host_*

Notes:

- SSH Host Key is not mandatory to restore in every environment
- If performing a full migration of GitLab domain and SSH address, it's recommended to retain
- If not restored, users may need to reconfirm SSH fingerprints on first connection

---

## SixI don't know.Offsite Backup File Storage

Local backups cannot replace disaster recovery.

If the GitLab server's disk is damaged, local backups may also be lost.

Recommended backup storage strategy:

    GitLab generates backups locally
       |
       v
    Temporarily save in backup directory
       |
       v
    rsync / scp synchronize to backup server
       |
       v
    Regularly clean old backups
       |
       v
    Regular recovery drills

Example directory plan:

    /backup/gitlab/
    ├── app/
    ├── config/
    ├── ssl/
    └── ssh/

Create directories:

    sudo mkdir -p /backup/gitlab/app
    sudo mkdir -p /backup/gitlab/config
    sudo mkdir -p /backup/gitlab/ssl
    sudo mkdir -p /backup/gitlab/ssh

Copy application backup:

    sudo cp /var/opt/gitlab/backups/*_gitlab_backup.tar /backup/gitlab/app/

Copy configuration backup:

    sudo cp /etc/gitlab/config_backup/*.tar /backup/gitlab/config/

Copy certificate backup:

    sudo tar -czf /backup/gitlab/ssl/gitlab-ssl-$(date +%F).tar.gz /etc/gitlab/ssl

Copy SSH Host Key:

    sudo tar -czf /backup/gitlab/ssh/ssh-host-keys-$(date +%F).tar.gz /etc/ssh/ssh_host_*

Synchronize to remote backup server:

    rsync -avz /backup/gitlab/ backup-user@10.0.0.100:/data/backup/gitlab/

You can also use scp:

    scp -r /backup/gitlab backup-user@10.0.0.100:/data/backup/

---

## SevenI don't know.Scheduled Backup Plan

### 7.1 Write Backup Script

Example script:

    sudo mkdir -p /opt/scripts

    sudo tee /opt/scripts/gitlab-backup.sh > /dev/null <<'EOF'
    #!/bin/bash

    set -euo pipefail

    BACKUP_BASE="/backup/gitlab"
    DATE_TIME="$(date +%F-%H%M%S)"

    APP_BACKUP_DIR="${BACKUP_BASE}/app"
    CONFIG_BACKUP_DIR="${BACKUP_BASE}/config"
    SSL_BACKUP_DIR="${BACKUP_BASE}/ssl"
    SSH_BACKUP_DIR="${BACKUP_BASE}/ssh"

    mkdir -p "${APP_BACKUP_DIR}" "${CONFIG_BACKUP_DIR}" "${SSL_BACKUP_DIR}" "${SSH_BACKUP_DIR}"

    echo "[INFO] Start GitLab application backup..."
    gitlab-backup create

    echo "[INFO] Start GitLab config backup..."
    gitlab-ctl backup-etc

    echo "[INFO] Copy latest application backup..."
    LATEST_APP_BACKUP="$(ls -t /var/opt/gitlab/backups/*_gitlab_backup.tar | head -n 1)"
    cp "${LATEST_APP_BACKUP}" "${APP_BACKUP_DIR}/"

    echo "[INFO] Copy latest config backup..."
    LATEST_CONFIG_BACKUP="$(ls -t /etc/gitlab/config_backup/*.tar | head -n 1)"
    cp "${LATEST_CONFIG_BACKUP}" "${CONFIG_BACKUP_DIR}/"

```bash
if [ -d /etc/gitlab/ssl ]; then
  echo "[INFO] Backup GitLab SSL directory..."
  tar -czf "${SSL_BACKUP_DIR}/gitlab-ssl-${DATE_TIME}.tar.gz" /etc/gitlab/ssl
fi

echo "[INFO] Backup SSH host keys..."
tar -czf "${SSH_BACKUP_DIR}/ssh-host-keys-${DATE_TIME}.tar.gz" /etc/ssh/ssh_host_*

echo "[INFO] Clean old local backups, keep 14 days..."
find "${BACKUP_BASE}" -type f -mtime +14 -delete

echo "[INFO] GitLab backup finished."
EOF

Add execute permissions:

    sudo chmod +x /opt/scripts/gitlab-backup.sh

Manual test:

    sudo /opt/scripts/gitlab-backup.sh

---

### 7.2 Configure crontab

Backup daily at 2 AM:

    sudo crontab -e

Add:

    0 2 * * * /opt/scripts/gitlab-backup.sh >> /var/log/gitlab-backup.log 2>&1

Check logs:

    sudo tail -f /var/log/gitlab-backup.log

Check scheduled tasks:

    sudo crontab -l

---

## EightI don't know.GitLab Cross-Server Recovery Process

### 8.1 Pre-Recovery Preparation

Assume you want to restore GitLab to a new server.

Before recovery, must confirm:

- New server has GitLab installed
- GitLab version matches the source
- GitLab type is consistent (CE for CE, EE for EE)
- New server has sufficient disk space
- Backup files are copied to the new server
- Configuration files and secrets files are prepared
- External domain names, certificates, and access addresses are planned

Check current GitLab version:

    sudo gitlab-rake gitlab:env:info

---

### 8.2 Install Same Version GitLab

Example:

    sudo apt update

Specific installation version depends on the original GitLab version.

After installation, run at least once:

    sudo gitlab-ctl reconfigure

Confirm GitLab base services are running:

    sudo gitlab-ctl status

---

### 8.3 Restore /etc/gitlab Configuration

First back up the default configuration on the new machine:

    sudo mv /etc/gitlab /etc/gitlab.bak.$(date +%F-%H%M%S)

Extract original machine configuration backup:

    sudo tar -xf gitlab_config_1710000000_2026_04_28.tar -C /

Confirm files exist:

    sudo ls -l /etc/gitlab/gitlab.rb
    sudo ls -l /etc/gitlab/gitlab-secrets.json

Recommended permissions:

    sudo chmod 600 /etc/gitlab/gitlab-secrets.json

Reload configuration:

    sudo gitlab-ctl reconfigure

---

### 8.4 Place Application Backup Files

Copy application backup files to the backup directory:

    sudo cp 1710000000_2026_04_28_16.11.0_gitlab_backup.tar /var/opt/gitlab/backups/

Set permissions:

    sudo chown git:git /var/opt/gitlab/backups/1710000000_2026_04_28_16.11.0_gitlab_backup.tar
    sudo chmod 600 /var/opt/gitlab/backups/1710000000_2026_04_28_16.11.0_gitlab_backup.tar

Check:

    sudo ls -lh /var/opt/gitlab/backups/

---

### 8.5 Stop Related Services

Stop services that connect to the database before recovery:

    sudo gitlab-ctl stop puma
    sudo gitlab-ctl stop sidekiq

Check status:

    sudo gitlab-ctl status

Notes:

- Do not stop all services blindly after stopping
- Only stop critical services as required by GitLab recovery
- If the version is old, service names may differ (e.g. unicorn)

---

### 8.6 Execute Recovery

Backup file name:

    1710000000_2026_04_28_16.11.0_gitlab_backup.tar

Recovery command with only BACKUP identifier:

    sudo gitlab-backup restore BACKUP=1710000000_2026_04_28_16.11.0

During recovery, it will prompt to confirm overwriting database data, follow the prompt to confirm.

After recovery, execute:

    sudo gitlab-ctl reconfigure
    sudo gitlab-ctl restart

---

### 8.7 Post-Recovery Checks

Run GitLab check:

    sudo gitlab-rake gitlab:check SANITIZE=true

Check service status:

    sudo gitlab-ctl status

Check logs:

    sudo gitlab-ctl tail

Check web page:

    https://gitlab.example.com

Check projects:

- Does user exist?
- Does group exist?
- Does project exist?
- Is repository code complete?
- Are branches complete?
- Are Tags complete?
- Do Merge Requests exist?
- Can CI/CD Variables be read normally?
- Is Pipeline history present?
- Do Runners need to be re-registered?
- Are Webhooks normal?
- Is SSH Clone normal?
- Is HTTPS Clone normal?

Test Git clone:

    git clone git@gitlab.example.com:group/project.git

Or:

    git clone https://gitlab.example.com/group/project.git

---

## NineI don't know.GitLab Recovery Verification Checklist

After recovery, do not only check if the page can be opened.

Recommended to verify according to the following checklist:

| Check Item | Verification Method | Pass/Fail |
|---|---|---|
| GitLab Service Status | gitlab-ctl status |  |
| GitLab Self-Check | gitlab-rake gitlab:check SANITIZE=true |  |
| Web Login | Browser access to GitLab |  |
| Users & Groups | Check via Admin Area |  |
| Project List | Check critical projects |  |
| Git Clone | SSH/HTTPS clone test |  |
| Git Push | Test branch push |  |
| CI/CD Variables | Check variable availability |  |
| Pipeline | Manually trigger test pipeline |  |
| Runner | Check Runner online status |  |
| Webhook | Trigger test event |  |
| LFS | Check projects using LFS |  |
| Artifacts | Check pipeline artifacts |  |
| Package Registry | Check package repository |  |
| Container Registry | Check image repository (configuration-dependent) |  |
| Certificate | HTTPS access normal |  |
| SSH Fingerprint | No abnormal alerts on client connection |  |

---

## Ten. Common Issues and Troubleshooting

### 10.1 Version Mismatch During Restore

**Phenomenon:**

    GitLab version mismatch

**Reasons:**

- Backup source version differs from target GitLab version
- CE/EE type mismatch

**Resolution:**

    1. Check version information in backup filename
    2. Install GitLab version matching the backup
    3. Confirm CE/EE type consistency
    4. Re-execute the restore

---

### 10.2 Abnormal CI/CD Variables After Restore

**Possible Causes:**

- Failed to restore /etc/gitlab/gitlab-secrets.json
- secrets file overwritten by new instance
- Incorrect recovery sequence

**Resolution:**

    1. Restore gitlab-secrets.json from original GitLab
    2. Place back into /etc/gitlab/gitlab-secrets.json
    3. Execute gitlab-ctl reconfigure
    4. Execute gitlab-ctl restart

---

### 10.3 Runner Connection Failure

**Possible Causes:**

- GitLab URL change
- Token or secrets mismatch
- Runner registration information expired
- Network or certificate issues

**Troubleshooting:**

    sudo gitlab-ctl status
    sudo gitlab-ctl tail

**On Runner Node:**

    sudo gitlab-runner status
    sudo gitlab-runner verify

**Re-register Runner if needed:**

    sudo gitlab-runner register

---

### 10.4 Git Clone Reports SSH Fingerprint Change

**Phenomenon:**

    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

**Possible Causes:**

- New server SSH Host Key differs from old server

**Resolution:**

    1. If normal migration, restore old server /etc/ssh/ssh_host_* files
    2. Or notify users to clean known_hosts and re-verify fingerprint

**Client Cleanup Example:**

    ssh-keygen -R gitlab.example.com

---

### 10.5 Backup File Too Large

**Possible Causes:**

- Large repository size
- Artifacts not cleaned
- Many LFS files
- Package Registry data
- Container Registry data

**Optimization Directions:**

    1. Set artifacts expiration time
    2. Clean unused projects
    3. Clean unused branches and Tags
    4. Govern large repositories
    5. Regularly clean backup directory
    6. Use object storage and tiered backup for large-scale instances

---

### 10.6 Backup Successful but Restore Failed

**Possible Causes:**

- Backup file corruption
- Incorrect backup file permissions
- Target GitLab version mismatch
- Target instance already has same-named repository data
- Secrets file not restored
- Backup excludes external object storage data

**Troubleshooting:**

    sudo ls -lh /var/opt/gitlab/backups
    sudo chown git:git /var/opt/gitlab/backups/*_gitlab_backup.tar
    sudo gitlab-rake gitlab:env:info
    sudo gitlab-rake gitlab:check SANITIZE=true
    sudo gitlab-ctl tail

---

## Eleven. Production Environment Backup Strategy Recommendations

### 11.1 Backup Frequency Recommendations

| Environment | Application Data Backup | Configuration Backup |Alien. Synchronization | Recovery Drill |
|---|---|---|---|---|
| Test Environment | Weekly | After configuration change | Optional | Optional |
| Small/Mid Production | Daily | Daily or after configuration change | Mandatory | Monthly or quarterly |
| Critical Production | Daily or higher frequency | Daily or after configuration change | Mandatory | Monthly |
| Large-Scale Production | Tiered strategy | Immediately after configuration change | Mandatory | Regular drills |

---

### 11.2 Backup Retention Policy

**Example:**

    Last 7 days: Daily retention
    Last 4 weeks: Weekly retention
    Last 6 months: Monthly retention

**Simple Cleanup Commands:**

    find /backup/gitlab/app -type f -mtime +30 -delete
    find /backup/gitlab/config -type f -mtime +30 -delete
    find /backup/gitlab/ssl -type f -mtime +30 -delete
    find /backup/gitlab/ssh -type f -mtime +30 -delete

**Note:** Production environments should not rely solely on deletion scripts. Combine with backup platforms, object storage lifecycle policies, or dedicated backup systems.

---

### 11.3 Backup Security Requirements

GitLab backups may include:

- Source code
- User data
- Project permission information
- CI/CD variables
- Token
- Secret
- Internal domain names
- Business configurations

Therefore, backup files must protect: /think

1. Restrict permissions for backup directories  
2. Use SSH / VPN / intranetLine for backup transmission  
3. Do not place backup files in web-accessible directories  
4. Do not store gitlab-secrets.json with application backups long-term  
5. Backup servers require access control and auditing  
6. Important backups should be encrypted  
7. Revoke backup server permissions promptly for departing personnel  

Example permissions:  

    sudo chown -R root:root /backup/gitlab  
    sudo chmod -R 700 /backup/gitlab  

---

## TwelveI don't know.GitLab Backup Recovery Flowchart  

    ┌──────────────────────┐  
    │      GitLab Production Instance  │  
    └──────────┬───────────┘  
               │  
               │ gitlab-backup create  
               v  
    ┌──────────────────────┐  
    │   Application Data Backup tar    │  
    │ repositories/database │  
    │ uploads/artifactsWait.   │  
    └──────────┬───────────┘  
               │  
               │ gitlab-ctl backup-etc  
               v  
    ┌──────────────────────┐  
    │   Configuration & Key Backup      │  
    │ gitlab.rb/secrets     │  
    └──────────┬───────────┘  
               │  
               │ rsync/scp  
               v  
    ┌──────────────────────┐  
    │      Offsite Backup Server    │  
    └──────────┬───────────┘  
               │  
               │ Disaster Recovery  
               v  
    ┌──────────────────────┐  
    │   New GitLab Same-Version Instance │  
    └──────────┬───────────┘  
               │  
               │ Restore Configuration + Restore Application Data  
               v  
    ┌──────────────────────┐  
    │      Post-Recovery Verification        │  
    │ clone/push/pipeline   │  
    └──────────────────────┘  

---

## ThirteenI don't know.Interview Answer Strategy  

If asked:  

    How to perform a full backup of GitLab?  

Can respond:  

    GitLab backups cannot only backup repository directories. In production, I would split into two parts:  
    The first part is application data backup, using gitlab-backup create, backing up databases, repositories, uploads, LFS, artifacts, packages, etc. Backup files are default stored in /var/opt/gitlab/backups.  
    The second part is configuration and key backup, mainly /etc/gitlab/gitlab.rb and gitlab-secrets.json, which can be generated with gitlab-ctl backup-etc.  
    Among these, gitlab-secrets.json is critical, as CI/CD Variables, 2FA, Tokens, etc., depend on it for decryption. Only backing up application data without secrets may result in decryption failures or Runner anomalies after recovery.  
    During recovery, prepare a same-version, same-type GitLab instance, place backup files in the backup directory, restore configuration then execute reconfigure, followed by gitlab-backup restore BACKUP=xxx, finally restart and verify via gitlab:check, clone, push, pipeline, Runner status.  
    In production, backups must be synchronized to offsite servers and regular recovery drills should be conducted to avoid having only backups without recovery capability.  

---

## FourteenI don't know.Production Notes  

1. GitLab backups must include both application data and configuration keys.  
2. gitlab-secrets.json is one of the critical files for successful recovery.  
3. GitLab version and type must match during recovery.  
4. Onsite backups do not equal disaster recovery; backups must be stored offsite.  
5. Configuration backups and application data backups should not be stored long-term in the same location.  
6. Recovery drills must be conducted after backups.  
7. Recovery verification cannot only check if the interface can open; it must also validate clone, push, pipeline, Runner, variables, Webhook.  
8. When using object storage, external PostgreSQL, external Redis, external Registry, confirm if corresponding data is included in the backup strategy.  
9. Backups must be performed before upgrading GitLab.  
10. Production environments should clearly define RPO and RTO.  

---

## FifteenI don't know.Understanding RPO and RTO  

RPO:  

    Recovery Point Objective, recovery point target.  
    Represents the maximum acceptable data loss period.  

Example: Backing up once daily at 2 AM, if a failure occurs at 5 PM, data loss may occur from 2 AM to 5 PM.  

RTO:  

    Recovery Time Objective, recovery time target.  
    Represents the time required to restore services after a failure.  

Example: If GitLab must be restored within 2 hours, prepare:  

    1. Backup files  
    2. Same-version installation package  
    3. Recovery documentation  
    4. Recovery scripts  
    5. Backup server  
    6. DNS or access switching plan  

Without recovery drills, it's difficult to guarantee RTO.  

---

## SixteenI don't know.Reference Documents  

- GitLab Docs: Back up GitLab  
- GitLab Docs: Restore GitLab  
- GitLab Docs: Backup and restore configuration on a Linux package installation https://gitlab.example.com