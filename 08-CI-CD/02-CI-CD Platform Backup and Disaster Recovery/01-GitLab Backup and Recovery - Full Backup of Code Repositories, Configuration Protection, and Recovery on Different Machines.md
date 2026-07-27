# 01-GitLab Backup and Recovery: Full Backup of Code Repositories, Configuration Protection, and Recovery on Different Machines

Recommended Path: 08-CI-CD/02-CI-CD Platform Backup and Disaster Recovery/01-GitLab Backup and Recovery: Full Backup of Code Repositories, Configuration Protection, and Recovery.md

Tags: #CI-CD #GitLab #Backup and Recovery #Disaster Recovery #Code Repositories #Operation and Maintenance #SRE

---

## I. Document Description

GitLab serves as the central hub for code assets in an enterprise's CI/CD system, typically containing the following key components:

- Git code repositories
- User, group, project, and permission settings
- Issues, Merge Requests, and wikis
- CI/CD Pipeline data
- CI/CD Variables
- Artifacts
- LFS files
- Uploads attachments
- Package Registry data
- Container Registry data, depending on the deployment method
- GitLab system configuration
- GitLab key files

In a production environment, a reliable GitLab backup should not merely involve backing up the Git repository directory. It must also include:

1. Application data backup
2. Configuration file backup
3. Key file backup
4. Certificate and SSH Host Key backup
5. Verification of recovery on different machines
6. Regular backup tasks
7. Offsite storage of backup files
8. Recovery drills

This document focuses on the full backup, configuration protection, recovery processes, and production considerations for GitLab deployed using Linux Package/Omnibus.

---

## II. Why Is GitLab Backup Necessary?

If GitLab experiences a failure or data loss, it will affect not only the code repositories but also the entire development and delivery process.

Common risks include:

- Server disk corruption
- Accidental deletion of the GitLab instance
- Database damage
- Erasure of projects or branches
- Failed GitLab upgrades
- Loss of CI/CD Variables
- Runner authentication issues
- Loss of certificates or key files
- Incomplete data during migration to a new server
- Data unavailability due to attacks or operational errors

GitLab plays a critical role in the CI/CD system as follows:

    Developers
       |
       v
    GitLab Code Repositories
       |
       v
    Jenkins / GitLab CI
       |
       v
    Harbor Image Repository
       |
       v
    Kubernetes Cluster

If GitLab is lost, subsequent processes such as Jenkins, Harbor, and Kubernetes deployment will be affected.

Therefore, GitLab backup is considered the top priority in the disaster recovery strategy for the CI/CD platform.

---

## III. GitLab Backup Objects

### 3.1 Application Data Backup Objects

GitLab application data backups typically include:

- Database data
- Git repositories
- Wiki repositories
- LFS objects
- Uploads attachments
- CI/CD Artifacts
- Pages data
- Packages data
- Terraform State, depending on the version and configuration
- Container Registry data, depending on the configuration and storage method

Common backup commands:

    sudo gitlab-backup create

In older versions or some documents, you might also see:

    sudo gitlab-rake gitlab:backup:create

In production environments, it is recommended to use the commands recommended in the official documentation for your current GitLab version.

---

### 3.2 Configuration File Backup Objects

For GitLab deployed using Linux Package, the core configuration directory is:

    /etc/gitlab

Key files include:

    /etc/gitlab/gitlab.rb
    /etc/gitlab/gitlab-secrets.json

Among them:

- gitlab.rb: The main configuration file for GitLab
- gitlab-secrets.json: Stores sensitive information such as database encryption keys, CI/CD variable encryption keys, and 2FA-related keys

If only application data is backed up and gitlab-secrets.json is not included, the following issues may occur during recovery:

- CI/CD Variables cannot be decrypted
- 2FA users will be unable to log in normally
- Runner authentication will fail
- Webhooks, Tokens, and sensitive fields will be incorrect
- Some encrypted data will not be recoverable

Configuration backup command:

    sudo gitlab-ctl backup-etc

The default output directory is:

    /etc/gitlab/config_backup/

---

### 3.3 Certificates and SSH Host Key

If GitLab uses HTTPS, it is also necessary to back up the certificate files, typically located in:

    /etc/gitlab/ssl/

If you plan to migrate the entire machine, it is recommended to back up the SSH Host Key as well:

    /etc/ssh/

This is because Git clients usually store the server's SSH fingerprint. If the SSH Host Key changes after migration, the client may display a warning message:

    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

While this is not necessarily a failure, it can affect```markdown
sudo cp /etc/gitlab/config_backup/*.tar /backup/gitlab/config/

Copy certificate backups:

    sudo tar -czf /backup/gitlab/ssl/gitlab-ssl-$(date +%F).tar.gz /etc/gitlab/ssl

Copy SSH host keys:

    sudo tar -czf /backup/gitlab/ssh/ssh-host-keys-$(date +%F).tar.gz /etc/ssh/ssh_host_*

Sync to remote backup server:

    rsync -avz /backup/gitlab/ backup-user@10.0.0.100:/data/backup/gitlab/

You can also use scp:

    scp -r /backup/gitlab backup-user@10.0.0.100:/data/backup/

---

## Section 7: Scheduled Backup Plan

### 7.1 Creating a Backup Script

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

    echo "[INFO] Starting GitLab application backup..."
    gitlab-backup create

    echo "[INFO] Starting GitLab config backup..."
    gitlab-ctl backup-etc

    echo "[INFO] Copying the latest application backup..."
    LATEST_APP_backup="$(ls -t /var/opt/gitlab/backups/*_gitlab_backup.tar | head -n 1)"
    cp "${LATEST_APP_BACKUP}" "${APP_BACKUP_DIR}/"

    echo "[INFO] Copying the latest config backup..."
    LATEST_CONFIG_BACKUP="$(ls -t /etc/gitlab/config_backup/*.tar | head -n 1)"
    cp "${LATEST CONFIGBackingUP}" "${CONFIG_BACKUP_DIR}/"

    if [ -d /etc/gitlab/ssl ]; then
      echo "[INFO] Backing up the GitLab SSL directory..."
      tar -czf "${SSL_BACKUP_DIR}/gitlab-ssl-${DATE_TIME}.tar.gz" /etc/gitlab/ssl
    fi

    echo "[INFO] Backing up SSH host keys..."
    tar -czf "${SSH_BACKUP_DIR}/ssh-host-keys-${DATE_TIME}.tar.gz" /etc/ssh/ssh_host_*

    echo "[INFO] Removing old local backups, keeping only the last 14 days..."
    find "${BACKUP_BASE}" -type f -mtime +14 -delete

    echo "[INFO] GitLab backup completed."
    EOF

Add execution permissions:

    sudo chmod +x /opt/scripts/gitlab-backup.sh

Test manually:

    sudo /opt/scripts/gitlab-backup.sh

---

### 7.2 Configuring crontab

Back up at 2 AM every day:

    sudo crontab -e

Add the following line:

    0 2 * * * /opt/scripts/gitlab-backup.sh >> /var/log/gitlab-backup.log 2>&1

Check the log:

    sudo tail -f /var/log/gitlab-backup.log

View scheduled tasks:

    sudo crontab -l
```

---

## Section 8: GitLab Recovery Process on a Different Server

### 8.1 Preparation Before Recovery

Assume you want to restore GitLab to a new server.

Before restoration, make sure:

- The new server has GitLab installed.
- The version of GitLab matches the one from the backup.
- The type of GitLab is consistent (e.g., CE to CE, EE to EE).
- The new server has sufficient disk space.
- The backup files have been copied to the new server.
- The configuration and secrets files are ready.
- External domains, certificates, and access addresses are planned properly.

Check the current version of GitLab:

    sudo gitlab-rake gitlab:env:info
```

---

### 8.2 Installing the Same Version of GitLab

Example:

    sudo apt update

The specific installation version depends on the original one.

After installation, run at least once:

    sudo gitlab-ctl reconfigure

Check if GitLab's basic services are running:

    sudo gitlab-ctl status
```

---

### 8.3 Restoring /etc/gitlab Configuration

First, back up the default configuration on the new machine:

    sudo mv /etc/gitlab /etc/gitlab.bak.$(date +%F-%H%M%S)

Unzip the original configuration```markdown
git clone https://gitlab.example.com/group/project.git

---

## Section 9: GitLab Recovery Verification Checklist

After recovery, it is not enough to just ensure that the page can be opened. It is recommended to verify according to the following checklist:

| Check Item | Verification Method | Passed or Failed |
|---------------|-------------------|---------------------|
| GitLab Service Status | gitlab-ctl status |         |
| GitLab Self-check | gitlab-rake gitlab:check SANITIZE=true |         |
| Web Login     | Access GitLab via browser       |         |
| Users and Groups | Check in the Admin Area        |         |
| Project List    | Verify key projects             |         |
| Git Clone      | Test SSH/HTTPS cloning          |         |
| Git Push       | Test pushing to a branch        |         |
| CI/CD Variables | Check if variables are available     |         |
| Pipeline       | Manually trigger the test pipeline  |         |
| Runner Status    | Check the online status of runners   |         |
| Webhook        | Trigger a test event once          |         |
| LFS             | Verify projects using LFS         |         |
| Artifacts      | Check pipeline outputs              |         |
| Package Registry | Verify the package repository       |         |
| Container Registry | Verify the image repository (depending on configuration) |         |
| Certificates     | Ensure normal HTTPS access        |         |
| SSH Fingerprint  | No abnormal alerts in client connections |         |

---

## Section 10: Common Issues and Troubleshooting

### 10.1 Inconsistent Versions During Recovery

**Phenomenon:**  
GitLab version mismatch.

**Cause:**  
- The version of the backup source is different from the target GitLab version.  
- The CE/EE type does not match.

**Solution:**  
1. Check the version information in the backup file name.  
2. Install the same version of GitLab as the backup.  
3. Ensure that the CE/EE type is consistent.  
4. Re-execute the recovery process.

---

### 10.2 Abnormal CI/CD Variables After Recovery

**Possible Causes:**  
- The /etc/gitlab/gitlab-secrets.json file was not restored.  
- The secrets file was overwritten by the new instance.  
- The recovery order of configurations was incorrect.

**Solution:**  
1. Restore the gitlab-secrets.json file from the original GitLab installation.  
2. Place it back in /etc/gitlab/gitlab-secrets.json.  
3. Execute `gitlab-ctl reconfigure`.  
4. Execute `gitlab-ctl restart`.

---

### 10.3 Unable to Connect to Runner

**Possible Causes:**  
- The GitLab URL has changed.  
- The Token or secrets are incorrect.  
- The Runner registration information is invalid.  
- Network or certificate issues.

**Troubleshooting Steps:**  
- Run `sudo gitlab-ctl status` and `sudo gitlab-ctl tail`.  
- On the Runner node, check `sudo gitlab-runner status` and `sudo gitlab-runner verify`.  
- If necessary, re-register the Runner by running `sudo gitlab-runner register`.

---

### 10.4 SSH Fingerprint Change Reported During Git Clone

**Phenomenon:**  
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

**Possible Cause:**  
The SSH Host Key of the new server is different from that of the old server.

**Solution:**  
1. If it is a normal migration, restore the /etc/ssh/ssh_host_* files from the old server.  
2. Or ask users to clear their `known_hosts` file and re-verify the fingerprint.

**Example for clearing on the client side:**  
`ssh-keygen -R gitlab.example.com`

---

### 10.5 The Backup File Is Too Large

**Possible Causes:**  
- The repository size is large.  
- Unnecessary Artifacts are not cleared.  
- There are many LFS files.  
- The Package Registry contains a lot of data.  
- The Container Registry has a substantial amount of data.

**Optimization Steps:**  
1. Set an expiration time for Artifacts.  
2. Clean up unused projects.  
3. Remove unnecessary branches and Tags.  
4. Manage large repositories properly.  
5. Regularly clean the backup directory.  
6. For large-scale instances, consider using object storage and tiered backup solutions.

---

### 10.6 Successful Backup but Failed Recovery

**Possible Causes:**  
- The backup file is damaged.  
- The backup file does not have the correct permissions.  
- The target GitLab version is inconsistent.  
- The target instance already contains a repository with the same name.  
- The secrets file was not restored.  
- The backup does not include external object storage data.

**Trou### Part One: Application Data Backup
Use `gitlab-backup create` to back up data such as databases, repositories, attachments, LFS, artifacts, and packages. By default, backup files are stored in `/var/opt/gitlab/backups`.

### Part Two: Configuration and Secrets Backup
The main targets for backup are `/etc/gitlab/gitlab.rb` and `gitlab-secrets.json`. You can use `gitlab-ctl backup-etc` to generate configuration backups.

`gitlab-secrets.json` is particularly critical because sensitive data such as CI/CD Variables, 2FA codes, and Tokens rely on it for decryption. Only application data should be backed up; failing to include `gitlab-secrets.json` may result in variables being unable to be decrypted or Runner failures during recovery.

To perform a recovery, prepare a GitLab instance of the same version and type as the original. Place the backup files in the designated directory, restore the configuration by executing `reconfigure`, then use `gitlab-backup restore BACKUP=xxx`. Finally, restart GitLab and verify that all functions such as `gitlab:check`, `clone`, `push`, `pipeline`, and Runner are working correctly.

In a production environment, it is also necessary to synchronize backups to remote machines and conduct regular recovery drills to ensure that there is actually the capability to restore data in case of an outage.

---

## Chapter Fourteen: Production Considerations

1. GitLab backups must include both application data and configuration secrets.
2. `gitlab-secrets.json` is one of the key files for successful recovery.
3. The version and type of GitLab used during recovery must match the original.
4. Local backups are not equivalent to disaster recovery solutions; backups should be stored off-site.
5. It is not recommended to store both configuration backups and application data backups in the same location.
6. After backing up, it is essential to conduct recovery drills.
7. Recovery verification should not only check if pages can be loaded but also verify functions such as `clone`, `push`, `pipeline`, Runner, Variables, and Webhooks.
8. When using object storage, external PostgreSQL, Redis, or Registry services, ensure that these data are included in the backup strategy.
9. Always back up GitLab before performing any upgrades.
10. The RPO (Recovery Point Objective) and RTO (Recovery Time Objective) should be clearly defined in a production environment.

---

## Chapter Fifteen: Understanding RPO and RTO

### RPO (Recovery Point Objective)
The Recovery Point Objective defines how much data loss is acceptable. For example, if backups are taken every midnight, a failure that occurs after 2 PM would result in the loss of data from midnight onward.

### RTO (Recovery Time Objective)
The Recovery Time Objective specifies how quickly services must be restored after a failure. To achieve a 2-hour RTO for GitLab, you need to have the following prepared in advance:
- Backup files
- Installation packages of the same version
- Recovery documentation and scripts
- A backup server
- DNS or access switching solutions

Without regular recovery drills, it is difficult to ensure that the RTO objective can be met.

---

## Chapter Sixteen: Reference Documents

- GitLab Docs: Back up GitLab
- GitLab Docs: Restore GitLab
- GitLab Docs: Backup and restore configuration on a Linux package installation